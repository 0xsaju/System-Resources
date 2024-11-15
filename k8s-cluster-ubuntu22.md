# Kubernetes Cluster Setup Guide - Ubuntu 22.04

## Prerequisites Configuration (On All Nodes)

### Update System and Set Hostnames

```bash
# On master node
sudo hostnamectl set-hostname master
# On worker1
sudo hostnamectl set-hostname cluster1
# On worker2
sudo hostnamectl set-hostname cluster2
```

### Update `/etc/hosts` on all nodes

```bash
sudo nano /etc/hosts
```

Add these lines:
```
192.168.60.101 master
192.168.60.102 cluster1
192.168.60.103 cluster2
```

### Disable swap on all nodes

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Load required kernel modules

```bash
cat << EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure kernel parameters

```bash
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Install Container Runtime (On All Nodes)

### Install containerd

```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Update containerd config to use SystemdCgroup
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Install Kubernetes Components (On All Nodes)

### Add Kubernetes repository

```bash
# Add Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Kubernetes components

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Initialize Master Node (On Master Only)

### Initialize the cluster

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.60.101
```

### Set up kubectl for the root user

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install Calico network plugin

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

### Get join command

```bash
kubeadm token create --print-join-command
```

## Join Worker Nodes (On Worker Nodes)

### Run the join command obtained from the master node

```bash
# Example (actual command will be different)
sudo kubeadm join 192.168.60.101:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Verify Cluster Setup (On Master Node)

### Check node status

```bash
kubectl get nodes
```

Expected output:
```
NAME      STATUS   ROLES           AGE     VERSION
master    Ready    control-plane   5m      v1.29.x
cluster1  Ready    <none>          3m      v1.29.x
cluster2  Ready    <none>          3m      v1.29.x
```

## Troubleshooting

If nodes are not showing as Ready:

- Check pod status: `kubectl get pods -n kube-system`
- Check logs: `journalctl -xeu kubelet`
- Ensure containerd is running: `systemctl status containerd`
- Verify network connectivity between nodes

## Single Line Kubernetes Installation

For a quick and easy Kubernetes installation, you can use the following single line command to set up a K3s cluster:

### Master Node Setup

```bash
sudo ufw disable && curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--tls-san 192.168.10.10 --write-kubeconfig=/home/user/.kube/config --write-kubeconfig-mode=644 --disable traefik' sh -
```

This command will:

- Disable UFW (Uncomplicated Firewall).
- Download and install K3s, a lightweight Kubernetes distribution.
- Configure K3s with the specified options:
    - `--tls-san 192.168.10.10`: Add a Subject Alternative Name for the Kubernetes API server.
    - `--write-kubeconfig=/home/user/.kube/config`: Write the kubeconfig file to the specified path.
    - `--write-kubeconfig-mode=644`: Set the permissions for the kubeconfig file.
    - `--disable traefik`: Disable the default Traefik ingress controller.

After running this command, your K3s cluster will be up and running with the specified configuration.

### Worker Node Setup

Find the token using:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Then run:

```bash
sudo ufw disable && curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.10:6443 K3S_TOKEN=<TOKEN> sh -
```
