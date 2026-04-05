# Compute Node Setup Script

A bash script that automates the configuration of HPC compute nodes, connecting them to a PBS Pro head node in a CentOS 7 cluster environment.

---

## Prerequisites

- CentOS 7 compute node (booted and accessible)
- Head node running PBS Pro with:
  - A configured `/ddn` NFS shared directory
  - `pbspro-server` installed and running
  - `user1` account already created
- CentOS DVD/ISO mounted at `/dev/sr0` on the compute node
- PBS Pro 19.1.3 RPM located at `/pbspro_19.1.3.centos_7/` on the compute node
- Root access on both nodes

---

## Usage

Run the script **directly on the compute node** as root:

```bash
./setup_compute_node.sh <node_name> <headnode_ip> <headnode_password>
```

### Arguments

| Argument            | Description                                      | Example           |
|---------------------|--------------------------------------------------|-------------------|
| `node_name`         | Unique hostname for this compute node            | `node01`          |
| `headnode_ip`       | IP address of the head node                      | `192.168.56.100`  |
| `headnode_password` | Root password of the head node                   | `mypassword`      |

### Example

```bash
./setup_compute_node.sh node01 192.168.56.100 mypassword
```

---

## What the Script Does

The script automates the following steps in order:

### 1. Repository Setup
- Mounts the CentOS DVD at `/cdrom`
- Configures `CentOS-Media.repo` to use the local DVD as a package source
- Disables the online `CentOS-Base.repo` to avoid network dependency

### 2. Hostname Configuration
- Sets the node's hostname using `hostnamectl`

### 3. `/etc/hosts` Synchronization
- Fetches the current `/etc/hosts` from the head node
- Appends this node's IP and hostname
- Pushes the updated file back to the head node (keeping both in sync)

### 4. SSH Passwordless Access
Configures **bidirectional** passwordless SSH for both `root` and `user1`:

- Generates RSA key pairs on the compute node (if not already present)
- Adds the head node's public key to the compute node's `authorized_keys`
- Copies the compute node's public key to the head node via `ssh-copy-id`
- Creates `user1` (password: `123`) and sets up equivalent bidirectional SSH for that account

### 5. NFS Mount
- Installs `nfs-utils`
- Disables `firewalld`
- Creates `/ddn` directory
- Adds a persistent NFS mount entry to `/etc/fstab`:
  ```
  <headnode_ip>:/ddn  /ddn  nfs  defaults  0  0
  ```
- Mounts the share immediately with `mount -a`

### 6. PBS Pro Execution Daemon
- Disables SELinux (required for PBS)
- Installs `pbspro-execution-19.1.3-0.x86_64.rpm` from the local directory
- Installs `environment-modules`
- Sets `PBS_SERVER=headnode` in `/etc/pbs.conf`
- Writes MOM client config: `$clienthost headnode` → `/var/spool/pbs/mom_priv/config`
- Starts and enables the PBS service

### 7. Node Registration
- Runs `qmgr` on the head node to register the new compute node
- Assigns the node to the `hpcc` queue

---

## Adding More Compute Nodes

Re-run the script for each additional node with a unique name:

```bash
./setup_compute_node.sh node02 192.168.56.100 mypassword
./setup_compute_node.sh node03 192.168.56.100 mypassword
```

---

## Verifying the Setup

After the script completes, verify from the **head node**:

```bash
# Check node status
pbsnodes -a

# List all nodes
qmgr -c "list node"

# Confirm NFS mount on compute node
ssh root@<node_ip> "df -h /ddn"

# Test passwordless SSH
ssh root@<node_ip> "hostname"
ssh user1@<node_ip> "hostname"
```

---

## Notes

- **SELinux** is permanently disabled; a reboot may be required for the change to fully take effect.
- **`user1`** is created with password `123` — change this in production environments.
- The head node password is passed as a plain-text argument and used via `expect`. Avoid this on shared terminals; consider using SSH keys or a secrets manager instead.
- The script assumes the PBS RPM and related files are pre-staged on the compute node at `/pbspro_19.1.3.centos_7/`.

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `mount: wrong fs type` | `nfs-utils` not installed | Run `yum install -y nfs-utils` manually |
| PBS node shows `down` | PBS MOM not started | Run `service pbs restart` on the compute node |
| SSH still prompts for password | Key not copied correctly | Manually run `ssh-copy-id root@<headnode_ip>` |
| `/ddn` not mounted | NFS service not running on head node | Run `service nfs restart` on the head node |
| `yum` fails with no packages | DVD not mounted | Run `mount /dev/sr0 /cdrom` manually |
