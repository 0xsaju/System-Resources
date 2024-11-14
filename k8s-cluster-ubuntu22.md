Kubernetes Cluster Setup Guide - Ubuntu 22.04
Prerequisites Configuration (On All Nodes)

Update System and Set Hostnames

bashCopy# On master node
sudo hostnamectl set-hostname master
# On worker1
sudo hostnamectl set-hostname cluster1
# On worker2
sudo hostnamectl set-hostname cluster2

Update /etc/hosts on all nodes

bashCopysudo nano /etc/hosts

# Add these lines
192.168.60.101 master
192.168.60.102 cluster1
192.168.60.103 cluster2

Disable swap on all nodes

bashCopysudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

Load required kernel modules

bashCopycat << EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

Configure kernel parameters

bashCopycat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
Install Container Runtime (On All Nodes)

Install containerd

bashCopy# Install prerequisites
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
Install Kubernetes Components (On All Nodes)

Add Kubernetes repository

bashCopy# Add Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

Install Kubernetes components

bashCopysudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
Initialize Master Node (On Master Only)

Initialize the cluster

bashCopysudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.60.101

Set up kubectl for the root user

bashCopymkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Install Calico network plugin

bashCopykubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

Get join command

bashCopykubeadm token create --print-join-command
Join Worker Nodes (On Worker Nodes)

Run the join command obtained from the master node

bashCopy# Example (actual command will be different)
sudo kubeadm join 192.168.60.101:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Verify Cluster Setup (On Master Node)

Check node status

bashCopykubectl get nodes
Expected output:
CopyNAME      STATUS   ROLES           AGE     VERSION
master    Ready    control-plane   5m      v1.29.x
cluster1  Ready    <none>          3m      v1.29.x
cluster2  Ready    <none>          3m      v1.29.x
Troubleshooting
If nodes are not showing as Ready:

Check pod status: kubectl get pods -n kube-system
Check logs: journalctl -xeu kubelet
Ensure containerd is running: systemctl status containerd
Verify network connectivity between nodes