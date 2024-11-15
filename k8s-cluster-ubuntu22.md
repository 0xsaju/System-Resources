# Comprehensive Kubernetes Cluster Deployment Guide
## Ubuntu 22.04 LTS

This guide provides step-by-step instructions for setting up a production-ready Kubernetes cluster using Ubuntu 22.04 LTS. The setup includes one master node and multiple worker nodes using containerd as the container runtime.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Initial System Configuration](#initial-system-configuration)
- [Container Runtime Installation](#container-runtime-installation)
- [Kubernetes Installation](#kubernetes-installation)
- [Master Node Setup](#master-node-setup)
- [Worker Node Setup](#worker-node-setup)
- [Verification Steps](#verification-steps)
- [Alternative: Quick K3s Setup](#alternative-quick-k3s-setup)
- [Troubleshooting Guide](#troubleshooting-guide)

## Prerequisites

### Hardware Requirements
- Master Node: 2 CPU cores, 2GB RAM minimum
- Worker Nodes: 1 CPU core, 2GB RAM minimum
- All nodes must have Ubuntu 22.04 LTS installed
- Stable network connectivity between all nodes

### Network Requirements
- Unique hostname for each node
- Static IP addresses for all nodes
- Open ports:
  - Master Node: 6443, 2379-2380, 10250-10252
  - Worker Nodes: 10250, 30000-32767

## Initial System Configuration

### 1. Update System and Set Hostnames
Run on each node with appropriate hostname:
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Set hostname according to node role
# On master node:
sudo hostnamectl set-hostname k8s-master
# On worker nodes:
sudo hostnamectl set-hostname k8s-worker-1  # For first worker
sudo hostnamectl set-hostname k8s-worker-2  # For second worker
```

### 2. Configure Host Resolution
Add to `/etc/hosts` on all nodes:
```bash
sudo tee -a /etc/hosts <<EOF
192.168.60.101 k8s-master
192.168.60.102 k8s-worker-1
192.168.60.103 k8s-worker-2
EOF
```

### 3. Disable Swap
Execute on all nodes:
```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
```

### 4. Configure Kernel Modules
Execute on all nodes:
```bash
# Load required modules
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

# Apply modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure kernel parameters
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters
sudo sysctl --system
```

## Container Runtime Installation

### Install and Configure containerd
Execute on all nodes:
```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg software-properties-common

# Install containerd
sudo apt-get install -y containerd

# Create default configuration
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Kubernetes Installation

### Install Kubernetes Components
Execute on all nodes:
```bash
# Add Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes components
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Master Node Setup

### Initialize Kubernetes Cluster
Execute only on master node:
```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.60.101

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install Network Plugin (Calico)
Execute on master node:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

## Worker Node Setup

### Join Workers to Cluster
1. On master node, generate join command:
```bash
kubeadm token create --print-join-command
```

2. Execute the generated command on each worker node:
```bash
sudo kubeadm join 192.168.60.101:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

## Verification Steps

### Verify Cluster Status
Execute on master node:
```bash
# Check node status
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
```

Expected output for nodes:
```
NAME          STATUS   ROLES           AGE    VERSION
k8s-master    Ready    control-plane   5m     v1.29.x
k8s-worker-1  Ready    <none>          3m     v1.29.x
k8s-worker-2  Ready    <none>          3m     v1.29.x
```

## Alternative: Quick K3s Setup

For a lightweight alternative, you can use K3s:

### Master Node
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--tls-san 192.168.10.10 --write-kubeconfig=/home/user/.kube/config --write-kubeconfig-mode=644 --disable traefik' sh -
```

### Worker Nodes
```bash
# Get token from master
TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)

# Join worker
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.10:6443 K3S_TOKEN=$TOKEN sh -
```

## Troubleshooting Guide

### Common Issues and Solutions

1. **Nodes Not Ready**
   - Check kubelet status: `systemctl status kubelet`
   - View kubelet logs: `journalctl -xeu kubelet`
   - Verify network plugin pods: `kubectl get pods -n kube-system`

2. **Network Issues**
   - Verify node connectivity: `ping <node-ip>`
   - Check required ports: `netstat -plnt`
   - Review calico pods: `kubectl get pods -n calico-system`

3. **Pod Network Issues**
   - Check CNI configuration: `ls /etc/cni/net.d/`
   - Verify calico installation: `kubectl get pods -n calico-system`
   - Review pod logs: `kubectl logs -n calico-system <calico-pod-name>`

### Reset Cluster
If you need to start over:
```bash
# On master
sudo kubeadm reset
# On workers
sudo kubeadm reset
# Then remove configuration
rm -rf ~/.kube
```

For additional help or specific issues, consult the [official Kubernetes documentation](https://kubernetes.io/docs/setup/).