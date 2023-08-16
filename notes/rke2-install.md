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
# Copy contents from fileserver: array/storage/homesys/rke2/vals/mnt-fileserver.mount
sudo systemctl enable mnt-fileserver.mount
sudo nano /etc/systemd/system/mnt-fileserver.automount
# Copy contents from fileserver: array/storage/homesys/rke2/vals/mnt-fileserver.automount
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

### 2. Install Kubernetes
#### Core Components

Install RKE2
```bash
sudo mkdir -p /etc/rancher/rke2
# Copying config from fileserver
sudo cp /mnt/fileserver/rke2/vals/rke2-config.yaml /etc/rancher/rke2/config.yaml
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION=v1.26.6+rke2r1 sh -
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
sudo systemctl status rke2-server.service
```

Enable use of kubectl
```bash
mkdir .kube
sudo cp /etc/rancher/rke2/rke2.yaml .kube/config
sudo chown -R homesys:homesys .kube/
sudo nano /etc/profile
# Add this to bottom of file, then log out and back in
export PATH="/var/lib/rancher/rke2/bin:$PATH"
```

Install Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Install Cert-Manager, External-DNS, and Rancher charts
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

Install Cert-Manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.3/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.12.3
```

Apply resources from fileserver
```bash
kubectl apply -f /mnt/fileserver/rke2/main/01
kubectl apply -f /mnt/fileserver/rke2/secrets
kubectl apply -f /mnt/fileserver/rke2/main/02
kubectl apply -f /mnt/fileserver/rke2/configs
```

Install External-DNS
```bash
helm install external-dns bitnami/external-dns --namespace infra --set provider=cloudflare --set cloudflare.secretName=provider --set cloudflare.proxied=false --set replicaCount=1
```

Install Rancher
```bash
helm install rancher rancher-latest/rancher --create-namespace --namespace cattle-system -f /mnt/fileserver/rke2/vals/rancher.yaml
```

### 3. Continuous Deployment
#### Rancher UI

Once Rancher is done installing (confirm by checking status of deployments via `kubectl get pods -A -o wide`):
- Log into UI with bootstrap password from */mnt/fileserver/rke2/vals/rancher.yaml*
- Change password to the one in Bitwarden

#### VM SSH

Create SSH key pair
```bash
ssh-keygen -t ed25519 -C "gwyn@hannay.id.au"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Add key to GitHub (via GitHub UI)
```bash
cat .ssh/id_ed25519.pub
```

Create Secret for GitHub tokens
```bash
ssh-keyscan -H github.com
# Replace "output from keyscan" with the actual output from the above command, keep the double quotes
kubectl create secret generic ssh-key -n fleet-local --from-file=ssh-privatekey=/home/homesys/.ssh/id_ed25519 --from-literal=known_hosts="output from keyscan" --type=kubernetes.io/ssh-auth
```

Add GitRepo
```bash
kubectl apply -f /mnt/fileserver/rke2/main/03
```

Monitor rollout in Rancher UI
