# K8s Lab — Architecture

## Overview

A desktop-hosted, multi-node Kubernetes platform lab built on Hyper-V and operated from WSL2. Uses Ansible for node configuration, kubeadm for cluster bootstrap, and Cilium for eBPF networking.

## Host Environment

| Component | Detail |
|-----------|--------|
| CPU | Intel i7-13700K |
| RAM | 32 GB |
| Storage | 1 TB+ |
| Hypervisor | Hyper-V (Windows 11 Pro) |
| Operator Workstation | WSL2 (Ubuntu 24.04) |

## Network Layout

| Host | Role | IP |
|------|------|----|
| Windows Host | Hyper-V / WSL2 gateway | 192.168.0.253 |
| WSL2 | Operator workstation | 172.19.67.153 (NAT) |
| k8s-control | Control plane | 192.168.0.210 |
| k8s-worker-01 | Worker node | 192.168.0.211 |
| k8s-worker-02 | Worker node | 192.168.0.212 |

## VM Topology

```
Windows 11 Pro (i7-13700K, 32GB RAM)
├── Hyper-V
│   ├── k8s-template   (OFF — golden image, 2 vCPU, 2GB RAM, 20GB disk)
│   ├── k8s-control    (2 vCPU, 4GB RAM, 40GB disk)
│   ├── k8s-worker-01  (2 vCPU, 6GB RAM, 40GB disk)
│   └── k8s-worker-02  (2 vCPU, 6GB RAM, 40GB disk)
└── WSL2 (Ubuntu 24.04)
    ├── kubectl
    ├── helm
    ├── ansible
    └── git
```

## Kubernetes Cluster

| Component | Choice | Reason |
|-----------|--------|--------|
| Bootstrap | kubeadm | Full control over cluster init |
| Container Runtime | containerd (Docker repo) | CRI-compliant, avoids stale Ubuntu package |
| CNI | Cilium 1.16.0 | eBPF-based, production-relevant |
| Cgroup Driver | systemd | Required for modern Ubuntu + kubelet alignment |

## Networking Details

- Pod CIDR: `10.244.0.0/16`
- Control plane endpoint: `192.168.0.210:6443`
- WSL2 → LAN route: `192.168.0.0/24 via 172.19.64.1`
- Route persisted via `/etc/wsl.conf` boot command

## SSH Access

- Key type: ed25519
- Key name: `k8s_lab`
- Public key location on nodes: `~/.ssh/authorized_keys`
- All nodes accessible passwordlessly from WSL2
