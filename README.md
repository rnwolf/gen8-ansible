# HP ProLiant Gen8 Home Server Automation

## Project Purpose
A reproducible, high-security Ubuntu 26.04 environment for large-scale data ingest and collaborative Git hosting (Forgejo) . 

## Container Philosophy
We use **Podman** over Docker for better integration with the Linux kernel and UFW. 
Services are defined using **Quadlets** (systemd-native container units) located in `/etc/containers/systemd/`. 
The reverse proxy is **Caddy**, chosen for its native TLS handling and simpler rate-limiting syntax compared to Nginx.

The Podman "Storage Graph" (where container layers live) should ideally stay on the SSD for speed, while your Forgejo volumes (the actual git data) should be mapped to your ZFS pool (/tank/gitea) for safety and snapshots.

### Understanding Quadlets (The Future of your Services)

Instead of a docker-compose.yml, we will create .container files. When we run your Ansible script, it will drop a file like forgejo.container into /etc/containers/systemd/, and systemd will automatically start, stop, and log the container like a real system service.

#### Why this is better:

Auto-Update: Podman can automatically pull new images and restart services if they pass a health check.

Clean Logs: You can use journalctl -u forgejo to see exactly what is happening.


## Hardware Configuration Logic
* **Chassis:** HP ProLiant MicroServer Gen8
* **CPU:** Xeon E3-1265L v2 (Upgraded from Celeron for ZFS and AES-NI support)
* **RAM:** 16GB ECC (Required for ZFS ARC Cache)
* **Bay 1:** 1TB SSD (System Drive / OS)
* **Bay 2:** 20TB Seagate Exos (Data Drive / ZFS Pool)

### Why we boot from Bay 1 (SSD)
The HP Gen8 BIOS has a hardcoded limitation: it can only natively boot from Bay 1 or Bay 2 if the integrated B120i controller is in AHCI mode.
* **The "SD Card Bridge" Failure:** We attempted to use an internal microSD card to "jump" the boot to an SSD in the ODD (Optical) bay. This proved unreliable with Ubuntu 26.04 due to modern EXT4 features (metadata checksums) that GRUB 2.06 binaries often fail to read.
* **The Decision:** For 100% boot reliability and reproducibility, the SSD is physically housed in **Bay 1**. This ensures the server always finds the bootloader without complex "bridge" configurations.

## Recovery Scenarios

### Scenario A: Replacing the System SSD
1.  Physically install a new SSD into Bay 1.
2.  Install base Ubuntu Server 26.04.
3.  Ensure your SSH key is authorized.  `ssh-copy-id -i ~/.ssh/id_rsa.pub user@remote_host`
4.  Run the Ansible `site.yml` playbook from your management laptop.
5.  **Manual Step:** Import the existing ZFS pool (`zpool import tank`).

### Scenario B: 20TB Drive Failure
1.  Replace the drive in Bay 2.
2.  Follow the manual ZFS creation steps (See "Destructive Tasks" below).
3.  Restore data from backup.

## Manual (Destructive) Tasks 20TB ZFS Pool Initialization

To prevent accidental data loss, the initialization of the data drive is **never automated**. Follow these steps once the OS is installed and the `baseline.yml` playbook has completed.

### 1. Identify the Disk

The Gen8 identifies the 20TB Exos usually as `/dev/sdb`, but we will use the **ID** to be safe against device name shifts.

```bash
ls -l /dev/disk/by-id/ | grep sdb

Output:
lrwxrwxrwx 1 root root  9 Apr 24 09:03 ata-ST20000NM007D-3DJ103_ZVT64HM8 -> ../../sdb
lrwxrwxrwx 1 root root 10 Apr 24 09:03 ata-ST20000NM007D-3DJ103_ZVT64HM8-part1 -> ../../sdb1

```

### 2. Wipe the 20TB disk

`sudo wipefs -a /dev/disk/by-id/ata-ST20000NM007D-3DJ103_ZVT64HM8`

Troubleshooting 'Device or Resource Busy':

> If wipefs fails, check lsblk for auto-mounted partitions or vgdisplay for legacy LVM volumes. Use vgchange -an <name> to release the lock before wiping.


### 3. Create the pool named 'tank' (a classic ZFS naming convention)

```bash
sudo zpool create -f -o ashift=12 \
    -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
    -O compression=lz4 -O atime=off -O canmount=off \
    tank /dev/disk/by-id/ata-ST20000NM007D-3DJ103_ZVT64HM8
```

### 4. Create Service-Specific Datasets

We do not store data in the root of the pool. We create datasets so we can snapshot them individually.

```bash

# General storage for ingests
sudo zfs create -o mountpoint=/storage tank/storage

# Dedicated Forgejo data (Git repos)
sudo zfs create -o mountpoint=/tank/forgejo tank/forgejo

# Set permissions for your user
sudo chown -R $(whoami):$(whoami) /storage /tank/forgejo

```

### 5. Verify ZFS health

```bash
zpool status
zfs list
```

ZFS handles its own mounting by default.

Unlike standard Linux filesystems (EXT4/XFS) that require entries in /etc/fstab, ZFS is "self-mounting." Because we set the mountpoint property during the zfs create command, the ZFS kernel module will automatically mount these every time the Gen8 server boots.


To "hide" the root pool and only show the datasets, we run:

`sudo zfs set canmount=off tank`

Usually, we keep the root pool unmounted (canmount=off) to avoid cluttering the root directory, just ensure you don't accidentally save files directly to /tank/ instead of /tank/forgejo/

# How to deploy publish key to remote Linux server

## PowerShell (Windows):

```powershell
# Variables
$User = "username"          # Remote username
$RemoteHost = "remote.server.com" # Remote host or IP
$PubKeyPath = "$HOME\.ssh\id_rsa.pub"

# Copy the public key to the remote server
ssh "$User@$RemoteHost"
mkdir -p ~/.ssh
chmod 700 ~/.ssh
vim ~/.ssh/authorized_keys  (add the public key to the bottom of the file.)
chmod 600 ~/.ssh/authorized_keys
```

## Bash (Linux):

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub user@remote_host
```

## Update variables for Ansible playbooks

vars/main.yml: Public/General variables (Email, Domain, DNS).
vars/secrets.yml: Encrypted variables (Cloudflare Token, passwords).

## Deployment Instructions
To restore the server from a blank SSD:
1. Install base Ubuntu Server 26.04.
2. If running ansible from WSL you will need to ensure that ansible user does not need to type password for escalating to sudo.
3. Run: `ansible-playbook -i inventory.ini site.yml --ask-vault-pass --ask-become-pass`

## Playbook Roles
- **baseline.yml**: Handles hardware-specific fixes (LVM) and the "Podman + Caddy" stack.
- **security.yml**: Enforces SSH-key-only access and configures the UFW firewall to protect the ZFS data.
