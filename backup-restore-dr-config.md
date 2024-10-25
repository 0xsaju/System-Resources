# DR Backup & Restore Procedure (Step-by-Step)

## Overview
This procedure outlines the steps to automate the backup of Proxmox Virtual Machine (VM) dumps from the primary server (pve-38) to a disaster recovery (Robi Cloud) server using rsync over SSH. The process includes:

- Using rsync to transfer backup files
- Utilizing SSH key-based authentication for secure access, restricted to the pve-38 server

## 1. Preparation

### 1.1. Create a Backup User on the DR Server
On the DR server, create a dedicated backup user named db-backup to perform the file transfers. This user will be restricted to accessing only the backup directory.

```bash
sudo adduser db-backup
sudo mkdir -p /proxmox_backup_38
sudo chown db-backup:db-backup /proxmox_backup_38
```

### 1.2. Set Up SSH Key-Based Authentication
Generate an SSH key pair on the Proxmox server (pve-38):

```bash
ssh-keygen -t rsa -b 4096 -C "pve-backup"
```

Copy the public key to the DR server:
```bash
ssh-copy-id db-backup@192.168.10.10
```

Verify key-based login:
```bash
ssh db-backup@192.168.10.10
```

## 2. Backup Script Setup

### 2.1. Backup Script (/var/db-backup/backup.sh)
```bash
#!/bin/bash

# Define variables
DUMP_DIR="/mnt/pve/local-extra-ssd-3/dump/"
DR_SERVER="db-backup@192.168.10.10:/proxmox_backup_38/"
LOG_FILE="/var/log/rsync_backup.log"
EMAIL="sysadmin@example.com"

# Sync only .vma.zst files to DR server using SSH
rsync -avz -e ssh --include='*/' --include='*.vma.zst' --exclude='*' $DUMP_DIR $DR_SERVER > $LOG_FILE 2>&1

# Check if rsync succeeded
if [ $? -eq 0 ]; then
    echo "PVE-38 Dump Backup successfully transferred to Robi Cloud at $(date)" | mail -s "PVE-38 DR Backup Successful" $EMAIL
else
    echo "PVE-38 Dump Backup failed at $(date). Check the log at $LOG_FILE for details." | mail -s "PVE-38 DR Backup Failed" $EMAIL
fi
```

### 2.2. Script Explanation
- `DUMP_DIR`: Directory on the Proxmox server where the VM dump files are stored
- `DR_SERVER`: DR server destination directory with the db-backup user
- `LOG_FILE`: Log file location to record the rsync output
- `EMAIL`: Email address to receive notifications about backup status
- The rsync command synchronizes only .vma.zst files, deleting any files on the DR server that no longer exist on the Proxmox server
- Depending on the rsync result, an email notification is sent to sysadmin@example.com

## 3. Automate the Backup Process

### 3.1. Crontab Entry on Proxmox Server
To schedule the backup script to run daily at 2:00 AM, add the following crontab entry:

```bash
# Edit crontab
crontab -e

# Add the following line
0 2 * * * /bin/bash /var/db-backup/backup.sh

# Verify crontab entry
crontab -l
0 2 * * * /bin/bash /var/db-backup/backup.sh
```

## 4. DR Server Directory Structure
On the DR server, the backup files are stored in:
```bash
/proxmox_backup_38/
```

## 5. Verification

### Test Backup Script
Manually run the script to ensure everything is working:
```bash
root@pve38:~# sh /var/db-backup/backup.sh
```

### Check Logs
Verify the contents of /var/log/rsync_backup.log for any errors or issues:
```bash
sending incremental file list
dump/
dump/vzdump-qemu-100-2024_03_20-02_00_01.vma.zst
dump/vzdump-qemu-101-2024_03_20-02_00_01.vma.zst
dump/vzdump-qemu-102-2024_03_20-02_00_01.vma.zst

sent 15.24M bytes  received 92 bytes  1.74M bytes/sec
total size is 152.89M  speedup is 10.03
```

### Email Confirmation
Example of successful backup email notification:
```
From: root <root@pve38.cefalo.local>
Subject: PVE-38 DR Backup Successful
To: sysadmin@example.com

PVE-38 Dump Backup successfully transferred to Robi Cloud at Wed Mar 20 02:00:01 UTC 2024
```

### Data Backup Verification
Verify the data backup on the DR server:
```bash
db-backup@dr-server:~$ ls -l /proxmox_backup_38/dump/
total 152892
-rw-r--r-- 1 db-backup db-backup 52428800 Mar 20 02:00 vzdump-qemu-100-2024_03_20-02_00_01.vma.zst
-rw-r--r-- 1 db-backup db-backup 52428800 Mar 20 02:00 vzdump-qemu-101-2024_03_20-02_00_01.vma.zst
-rw-r--r-- 1 db-backup db-backup 52428800 Mar 20 02:00 vzdump-qemu-102-2024_03_20-02_00_01.vma.zst
```

## 6. Restore Process

### 6.1. Preparation

#### 6.1.1. Identify the VMs to Restore
- Review the available backup files on the DR server
- VM dump files are saved in the .vma.zst format

On the DR server, list the backup files:
```bash
ssh db-backup@192.168.10.10
ls /proxmox_backup_38/
```

#### 6.1.2. Ensure Sufficient Storage
Ensure that the target Proxmox server (pve-38) has enough storage to accommodate the VM backup files you plan to restore.

#### 6.1.3. Verify Network Connectivity
Ensure network connectivity between the Proxmox and DR servers. Test SSH access if needed:
```bash
ssh db-backup@192.168.10.10
```

### 6.2. Transfer Backup Files from DR Server to Proxmox Server

#### 6.2.1. Retrieve Backup Files
Use rsync to transfer the VM dump file from the DR server to the Proxmox server:
```bash
rsync -avz -e ssh db-backup@192.168.10.10:/proxmox_backup_38/<vm-backup-file>.vma.zst /mnt/pve/local-extra-ssd-3/dump/
```
Replace `<vm-backup-file>` with the name of the VM backup file to be restored.

#### 6.2.2. Verify File Transfer
Ensure the file is successfully transferred to the Proxmox server:
```bash
ls /mnt/pve/local-extra-ssd-3/dump/
```

### 6.3. Restore VM on Proxmox Server

#### 6.3.1. Start the VM Restore
Use qmrestore to restore the VM from the backup file:
```bash
qmrestore /mnt/pve/local-extra-ssd-3/dump/<vm-backup-file>.vma.zst <vmid>
```
- Replace `<vm-backup-file>` with the name of the backup file
- Replace `<vmid>` with the Proxmox VM ID to which the VM should be restored

#### 6.3.2. Verify the Restoration Process
After the restoration completes, check the status of the restored VM:
```bash
qm status <vmid>
```

### 6.4. Post-Restoration Steps

#### 6.4.1. Start the Restored VM
```bash
qm start <vmid>
```

#### 6.4.2. Verify VM Functionality
Access the VM through the Proxmox interface to ensure it's functioning correctly.

#### 6.4.3. Check Logs
Review VM logs for any issues:
```bash
journalctl -xe
```

#### 6.4.4. Notify Stakeholders
Notify stakeholders about the successful restore and ensure that critical services are functioning.

### 6.5. Testing the Restore Process

#### 6.5.1. Periodic Test Restoration
It is recommended to periodically test the restore process to ensure backup integrity and procedure reliability. Restore a non-critical VM as a test.

#### 6.5.2. Data Integrity Check
After testing the restore, ensure that the restored data and configuration are correct by performing functional tests on the VM.