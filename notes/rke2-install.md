# Simple RKE2 Install

### 1. Setup VM
#### ESXi UI

Create new VM in ESXi:
- Name: homesys
- OS: Ubuntu Linux (64-bit)
- Storage: datastore
- Virtual hardware:
  - CPU: 4
  - Memory: 31GB
  - Hard disk 1: Maximum possible (*Note: VMWare requires as much free space as there is RAM for the swap file, so it's disk capacity minus RAM*)
  - Add hard disk: existing hard disk: tera (haven't tried this yet)

Install Ubuntu from ISO:
- Update to new installer
- Deselect "create disk as LVM group"
- Select to enable SSH, import SSH identity from GitHub

If the IP address of this VM is somehow different from previous, the NFS share needs to be edited on the fileserver to point to this VM. On fileserver:
- Edit `/etc/export` to replace IP address of old VM with new one
- Apply config change with `sudo exportfs -a`

#### VM SSH
Set timezone
```bash
sudo timedatectl set-timezone Australia/Perth
```

Disable swap
```bash
swapoff -a
# Comment out swap file here
sudo nano /etc/fstab
```

Prevent "too many files" issue
```bash
sudo nano /etc/sysctl.d/99-sysctl.conf
# Add this line to the end
fs.inotify.max_user_instances=512
# Then run:
sudo service procps force-reload
```

Install and update packages
```bash
sudo apt update
sudo apt upgrade
sudo apt install nfs-common
sudo apt autoremove
```

Create mounts
```bash
sudo mkdir -p /mnt/fileserver
sudo nano /etc/systemd/system/mnt-fileserver.mount
# Copy contents from fileserver
sudo systemctl enable mnt-fileserver.mount
sudo nano /etc/systemd/system/mnt-fileserver.automount
# Copy contents from fileserver
sudo systemctl enable mnt-fileserver.automount
sudo systemctl daemon-reload
sudo systemctl start mnt-fileserver.automount
```

Ensure correct access to fileserver
```bash
sudo useradd -u 1001 files
sudo adduser homesys files
sudo adduser files homesys
```
