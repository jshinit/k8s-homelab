# K8s Lab — Setup & Bootstrap

## Phase 1 — WSL2 Operator Workstation

### Install WSL2 and Ubuntu
```powershell
wsl --install -d Ubuntu-24.04
```

### Install Operator Tools
```bash
sudo apt update && sudo apt install -y curl git vim net-tools openssh-client python3-pip ansible

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Generate SSH Key
```bash
ssh-keygen -t ed25519 -C "k8s-lab" -f ~/.ssh/k8s_lab -N ""
```

### Persist WSL2 Route to LAN
Add to `/etc/wsl.conf`:
```ini
[boot]
systemd=true
command = ip route add 192.168.0.0/24 via 172.19.64.1

[user]
default=brooks
```

---

## Phase 2 — Template VM

### Hyper-V Setup
1. Create External Virtual Switch named `k8s-external` bound to physical NIC
2. Create VM: Generation 2, 2 vCPU, 2GB RAM, 20GB disk
3. Security → Secure Boot → change to "Microsoft UEFI Certificate Authority"
4. Boot from Ubuntu 24.04 Server ISO

### Ubuntu Install Settings
- Server name: `k8s-template`
- Username: `brooks`
- OpenSSH: enabled
- Snaps: none

### Post-Install Fix (SSH host keys missing)
```bash
sudo ssh-keygen -A
sudo systemctl enable ssh
sudo systemctl restart ssh
```

### Add WSL2 Public Key
```bash
mkdir -p ~/.ssh
echo "<your-public-key>" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Template Cleanup Before Cloning
```bash
sudo apt update && sudo apt upgrade -y
sudo cloud-init clean
sudo truncate -s 0 /etc/machine-id
history -c
sudo shutdown now
```

---

## Phase 3 — Clone and Configure Nodes

### Hyper-V Export and Import
1. Export `k8s-template` to a staging folder
2. Import 3 times using "Copy the virtual machine (create a new unique ID)"
3. Set destination folders per node to avoid VHDX conflicts:
   - VM config: `C:\ProgramData\Microsoft\Windows\Hyper-V\<nodename>`
   - VHD: `C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\<nodename>`
4. Rename imported VMs to `k8s-control`, `k8s-worker-01`, `k8s-worker-02`

### Adjust VM Resources
| VM | vCPU | RAM |
|----|------|-----|
| k8s-control | 2 | 4096 MB |
| k8s-worker-01 | 2 | 6144 MB |
| k8s-worker-02 | 2 | 6144 MB |

### Set Static IPs via Netplan
On each node edit `/etc/netplan/50-cloud-init.yaml`:
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.0.210/24   # change per node
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        - to: 0.0.0.0/0
          via: 192.168.0.1
```
```bash
sudo netplan apply
```

### Set Hostnames
```bash
sudo hostnamectl set-hostname k8s-control   # or worker-01 / worker-02
```

### Add /etc/hosts Entries (all 3 nodes)
```bash
echo "192.168.0.210 k8s-control
192.168.0.211 k8s-worker-01
192.168.0.212 k8s-worker-02" | sudo tee -a /etc/hosts
```

---

## Phase 4 — Ansible Node Preparation

### Inventory File (`~/k8s-lab/ansible/inventory.ini`)
```ini
[control]
k8s-control ansible_host=192.168.0.210

[workers]
k8s-worker-01 ansible_host=192.168.0.211
k8s-worker-02 ansible_host=192.168.0.212

[all:vars]
ansible_user=brooks
ansible_ssh_private_key_file=~/.ssh/k8s_lab
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

### Run Playbook
```bash
ansible-playbook -i ~/k8s-lab/ansible/inventory.ini ~/k8s-lab/ansible/k8s-prep.yml -K
```

### What the Playbook Does
- Disables swap permanently
- Loads overlay and br_netfilter kernel modules
- Sets sysctl values for IP forwarding and bridge traffic
- Installs containerd from Docker's repo (not Ubuntu default)
- Sets SystemdCgroup = true in containerd config
- Installs kubeadm, kubelet, kubectl v1.31
- Pins package versions with hold
- Installs conntrack

---

## Phase 5 — Cluster Bootstrap

### Initialize Control Plane
```bash
sudo kubeadm init \
  --control-plane-endpoint=192.168.0.210 \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.0.210
```

### Configure kubectl on Control Node
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Copy kubeconfig to WSL2
```bash
mkdir -p ~/.kube
scp -i ~/.ssh/k8s_lab brooks@192.168.0.210:~/.kube/config ~/.kube/config
```

### Join Workers
```bash
sudo kubeadm join 192.168.0.210:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```
> Token expires in 24 hours. Regenerate with:
> `sudo kubeadm token create --print-join-command`

### Install Cilium CNI
```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium --version 1.16.0 \
  --namespace kube-system \
  --set ipam.mode=kubernetes
```

### Verify Cluster
```bash
kubectl get nodes        # all 3 should show Ready
kubectl get pods -n kube-system   # all cilium pods Running
```

---

## Troubleshooting Log

### SSH host keys missing on template VM
- **Symptom:** `ssh.service` failed to start, status showed exit-code failure
- **Cause:** Host keys not generated during install
- **Fix:** `sudo ssh-keygen -A && sudo systemctl restart ssh`

### WSL2 cannot reach Hyper-V VMs
- **Symptom:** SSH connection refused from WSL2
- **Cause:** WSL2 is on NAT network (172.19.x.x), VMs are on LAN (192.168.0.x)
- **Fix:** `sudo ip route add 192.168.0.0/24 via 172.19.64.1`
- **Persist:** Add to `/etc/wsl.conf` boot command

### kubeadm preflight error: conntrack not found
- **Symptom:** `[ERROR FileExisting-conntrack]` during kubeadm init
- **Fix:** `ansible all -i inventory.ini -m apt -a "name=conntrack state=present" -b -K`

### Hyper-V import VHDX conflict
- **Symptom:** Error copying virtual hard disks to destination folder, file already exists
- **Cause:** Default import path conflicts with existing template VHDX
- **Fix:** Specify unique destination folders per node during import wizard
