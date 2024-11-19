# Container Networking: From Zero to Advanced

## Part 1: Networking Fundamentals

### 1.1 Basic Networking Concepts

#### The OSI Model and TCP/IP Stack
The OSI (Open Systems Interconnection) model is fundamental to understanding network communications. Let's break down each layer and its practical applications in container networking:

##### Layer 1 - Physical Layer
- **Definition**: Handles the physical transmission of raw bits over a physical medium
- **In Container Context**: While containers don't directly deal with physical layer, understanding physical network limitations is crucial for:
  - Network performance optimization
  - Troubleshooting network latency
  - Planning container deployments across different data centers

##### Layer 2 - Data Link Layer
- **Definition**: Handles node-to-node communication
- **Key Concepts**:
  - MAC Addresses
  - Frame structure
  - Error detection
- **In Container Context**:
  ```bash
  # View bridge interface MAC address
  ip link show docker0
  
  # Examine MAC addresses of containers
  docker exec container1 ip link show eth0
  ```

##### Layer 3 - Network Layer
- **Definition**: Handles routing between networks
- **Key Concepts**:
  - IP Addressing
  - Routing
  - Packet forwarding
- **Practical Example**:
  ```bash
  # Examine container IP routing table
  docker exec container1 ip route show
  
  # Check container IP address
  docker exec container1 ip addr show
  ```

##### Layer 4 - Transport Layer
- **Definition**: Handles end-to-end communication
- **Protocols**:
  - TCP: Connection-oriented, reliable
  - UDP: Connectionless, faster
- **Container Implementation**:
  ```bash
  # List listening ports in container
  docker exec container1 netstat -tulpn
  
  # Check established connections
  docker exec container1 ss -tunap
  ```

#### IP Addressing and Subnetting

##### IPv4 Addressing
- **Structure**: 32-bit address divided into 4 octets
- **Classes**:
  ```
  Class A: 0.0.0.0 to 127.255.255.255 (/8)
  Class B: 128.0.0.0 to 191.255.255.255 (/16)
  Class C: 192.0.0.0 to 223.255.255.255 (/24)
  ```

##### Subnet Calculation
- **CIDR Notation**:
  ```
  /24 = 256 addresses (255.255.255.0)
  /25 = 128 addresses
  /26 = 64 addresses
  /27 = 32 addresses
  ```

- **Practical Example**:
  ```bash
  # Configure container network with specific subnet
  docker network create --subnet=172.18.0.0/16 mynetwork
  
  # Inspect network configuration
  docker network inspect mynetwork
  ```

##### Network Planning Example
```bash
# Create multiple container networks with different subnets
docker network create --subnet=172.18.0.0/24 frontend
docker network create --subnet=172.18.1.0/24 backend
docker network create --subnet=172.18.2.0/24 database

# Attach containers to specific networks
docker run -d --name web --network frontend nginx
docker run -d --name api --network backend python-api
docker run -d --name db --network database postgres
```

#### DNS and Service Discovery

##### DNS Basics
- **Components**:
  - Name Servers
  - Records (A, AAAA, CNAME, MX)
  - Resolution Process

##### Container DNS Configuration
```bash
# Custom DNS configuration in Docker
docker run -d --dns=8.8.8.8 --dns-search=example.com nginx

# View container DNS configuration
docker exec container1 cat /etc/resolv.conf
```

##### Service Discovery Patterns
1. **DNS-based**:
   ```yaml
   # Docker Compose example with DNS
   version: '3'
   services:
     web:
       image: nginx
       networks:
         - frontend
     api:
       image: api-service
       networks:
         - frontend
         - backend
   
   networks:
     frontend:
     backend:
   ```

2. **Key-Value Store**:
   ```bash
   # Using Consul for service discovery
   docker run -d \
     --name=consul \
     -p 8500:8500 \
     consul agent -server -bootstrap -ui
   ```

#### Network Interfaces and Bridge Networking

##### Network Interface Types
1. **Physical Interfaces**:
   ```bash
   # List physical interfaces
   ip link show
   ```

2. **Virtual Interfaces**:
   ```bash
   # Create virtual interface pair
   ip link add veth0 type veth peer name veth1
   ```

3. **Bridge Interfaces**:
   ```bash
   # Create and configure bridge
   ip link add br0 type bridge
   ip link set br0 up
   ip addr add 192.168.1.1/24 dev br0
   ```

##### Bridge Configuration for Containers
```bash
# Create custom bridge network
docker network create \
  --driver bridge \
  --subnet=172.19.0.0/16 \
  --gateway=172.19.0.1 \
  custom-bridge

# Connect container to bridge
docker network connect custom-bridge container1

# Inspect bridge configuration
bridge link show
```

#### Linux Network Namespaces

##### Creating and Managing Namespaces
```bash
# Create network namespace
ip netns add blue
ip netns add red

# List namespaces
ip netns list

# Execute commands in namespace
ip netns exec blue ip addr show
```

##### Connecting Namespaces
```bash
# Create veth pair
ip link add veth0 type veth peer name veth1

# Move interfaces to namespaces
ip link set veth0 netns blue
ip link set veth1 netns red

# Configure interfaces
ip netns exec blue ip addr add 192.168.1.1/24 dev veth0
ip netns exec red ip addr add 192.168.1.2/24 dev veth1

# Bring interfaces up
ip netns exec blue ip link set veth0 up
ip netns exec red ip link set veth1 up
```

### 1.2 Linux Networking Basics

#### Network Interface Management

##### Interface Configuration
```bash
# Add IP address to interface
ip addr add 192.168.1.10/24 dev eth0

# Set interface up/down
ip link set eth0 up
ip link set eth0 down

# Configure interface properties
ip link set eth0 mtu 9000
```

##### Interface Monitoring
```bash
# Monitor interface statistics
ip -s link show eth0

# Watch interface traffic
watch -n 1 'ip -s link show eth0'
```

#### IP Tables and Network Security

##### Basic iptables Structure
```bash
# List current rules
iptables -L

# Add basic rules
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

##### Container Network Security
```bash
# Allow container communication
iptables -A FORWARD -i docker0 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o docker0 -j ACCEPT

# Restrict container access
iptables -A DOCKER-USER -i eth0 -j DROP
```

[Content continues with detailed explanations of Network Routing, further Network Namespace concepts, and Virtual Ethernet pairs...]


# Advanced Container and Kubernetes Networking Guide

## 1. Docker Networking Fundamentals

### Understanding Docker Networking
#### Default Network Drivers
```bash
# List available network drivers
docker info | grep -i network

# Default networks in Docker
docker network ls
```

Docker comes with several built-in network drivers:
- `bridge`: Default network driver
- `host`: Removes network isolation
- `none`: Disables networking
- `overlay`: Multi-host networking
- `macvlan`: Assigns MAC address to container

#### Network Driver Deep Dive
```bash
# Inspect bridge network
docker network inspect bridge

# View bridge interface in host
ip addr show docker0

# List network namespaces
ip netns list
```

### Creating Custom Docker Networks
#### Bridge Network Creation
```bash
# Create custom bridge network
docker network create \
  --driver bridge \
  --subnet=172.20.0.0/16 \
  --gateway=172.20.0.1 \
  --opt "com.docker.network.bridge.name"="custom-bridge" \
  custom-net

# Verify network creation
ip addr show custom-bridge
```

#### Advanced Network Configuration
```bash
# Create network with multiple subnets
docker network create \
  --driver bridge \
  --subnet=172.20.0.0/16 \
  --subnet=2001:db8::/64 \
  --gateway=172.20.0.1 \
  --ipv6 \
  dual-stack-net
```

### Hands-On Lab: Docker Networking
```bash
# 1. Create three different networks
docker network create frontend-net
docker network create backend-net
docker network create data-net

# 2. Deploy containers in different networks
docker run -d --name webapp --network frontend-net nginx
docker run -d --name api --network backend-net python:3.9
docker run -d --name db --network data-net postgres:13

# 3. Connect container to multiple networks
docker network connect backend-net webapp

# 4. Test connectivity
docker exec webapp ping api
```

## 2. Network Namespace Deep Dive

### Network Namespace Fundamentals
```bash
# Create network namespaces
ip netns add red
ip netns add blue
ip netns add green

# List network namespaces
ip netns list

# Execute commands in namespace
ip netns exec red ip link list
```

### Network Namespace Inspection and Management
```bash
# Inspect namespace details
ip netns exec red ip addr show
ip netns exec red ip route show
ip netns exec red netstat -tulpn

# Configure namespace interface
ip netns exec red \
  ip addr add 192.168.1.1/24 \
  dev veth-red
```

### Connecting Network Namespace to Host System
```bash
# Create veth pair
ip link add veth-host type veth peer name veth-ns

# Move one end to namespace
ip link set veth-ns netns red

# Configure interfaces
ip addr add 192.168.1.1/24 dev veth-host
ip netns exec red ip addr add 192.168.1.2/24 dev veth-ns

# Enable interfaces
ip link set veth-host up
ip netns exec red ip link set veth-ns up
```

### Root Network Namespace Integration
```bash
# View root namespace interfaces
ip link show

# Create bridge in root namespace
ip link add br0 type bridge
ip link set br0 up
ip addr add 192.168.1.254/24 dev br0

# Connect namespace to bridge
ip link set veth-host master br0
```

## 3. Advanced Networking Concepts

### Managing Egress Traffic Flow
```bash
# Configure NAT for namespace
ip netns exec red iptables -t nat -A POSTROUTING \
  -o veth-ns -j MASQUERADE

# Add default route
ip netns exec red ip route add default \
  via 192.168.1.254 dev veth-ns
```

### Connecting Multiple Custom Network Namespaces
```bash
# Create veth pairs between namespaces
ip link add veth-red-blue type veth \
  peer name veth-blue-red
ip link add veth-blue-green type veth \
  peer name veth-green-blue

# Move interfaces to respective namespaces
ip link set veth-red-blue netns red
ip link set veth-blue-red netns blue
ip link set veth-blue-green netns blue
ip link set veth-green-blue netns green

# Configure interfaces
ip netns exec red ip addr add 10.0.1.1/24 dev veth-red-blue
ip netns exec blue ip addr add 10.0.1.2/24 dev veth-blue-red
ip netns exec blue ip addr add 10.0.2.1/24 dev veth-blue-green
ip netns exec green ip addr add 10.0.2.2/24 dev veth-green-blue
```

### Configuring and Managing iptables Rules
```bash
# Basic iptables configuration in namespace
ip netns exec red iptables -A INPUT -p tcp --dport 80 -j ACCEPT
ip netns exec red iptables -A INPUT -p tcp --dport 443 -j ACCEPT
ip netns exec red iptables -P INPUT DROP

# Configure forwarding between namespaces
ip netns exec blue iptables -A FORWARD -i veth-blue-red -o veth-blue-green -j ACCEPT
ip netns exec blue iptables -A FORWARD -i veth-blue-green -o veth-blue-red -j ACCEPT
```

### Implementing Bridge Networking Between Namespaces
```bash
# Create bridge in separate namespace
ip netns add bridge-ns
ip link add br0 type bridge
ip link set br0 netns bridge-ns
ip netns exec bridge-ns ip link set br0 up

# Connect namespaces to bridge
ip link add veth-red-br type veth peer name veth-br-red
ip link set veth-red-br netns red
ip link set veth-br-red netns bridge-ns
ip netns exec bridge-ns ip link set veth-br-red master br0
```

### Inter-Process Communication Across Namespaces
```bash
# Create Unix domain socket in root namespace
ip netns exec red nc -l -U /tmp/ns-socket &

# Connect from another namespace
ip netns exec blue nc -U /tmp/ns-socket
```

### Understanding and Implementing FIB Network Topology
```bash
# View FIB entries
ip netns exec red ip route show table all

# Configure policy routing
ip netns exec red ip rule add from 192.168.1.0/24 table 100
ip netns exec red ip route add default via 192.168.1.254 table 100
```

## 4. Kubernetes Networking

### Infrastructure Setup for Kubernetes Cluster with Terraform
```hcl
# main.tf
provider "aws" {
  region = "us-west-2"
}

resource "aws_vpc" "k8s_vpc" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "kubernetes-vpc"
  }
}

resource "aws_subnet" "k8s_subnet" {
  vpc_id     = aws_vpc.k8s_vpc.id
  cidr_block = "10.0.1.0/24"
  
  tags = {
    Name = "kubernetes-subnet"
  }
}

resource "aws_instance" "k8s_master" {
  ami           = "ami-0123456789"
  instance_type = "t3.medium"
  subnet_id     = aws_subnet.k8s_subnet.id
  
  tags = {
    Name = "kubernetes-master"
  }
}
```

### Kubernetes Cluster Setup on AWS EC2 Instances
```bash
# Initialize master node
kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Join worker nodes
kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash <hash>
```

### Network Interfaces for Kubernetes Pod Communication with Bash CNI
```bash
#!/bin/bash
# Simple CNI script (cni.sh)

# Read network config from stdin
read -r config

# Extract container network namespace
netns="/proc/$K8S_POD_INFRA_CONTAINER_ID/ns/net"

# Create veth pair
ip link add pod-$RANDOM type veth peer name eth0

# Move eth0 to pod namespace
ip link set eth0 netns $netns

# Configure pod interface
ip netns exec $netns ip addr add $POD_IP/24 dev eth0
ip netns exec $netns ip link set eth0 up
```

### Dynamic IP Assignment for Kubernetes Pods with Bash CNI
```bash
#!/bin/bash
# IPAM script (ipam.sh)

IPAM_STORE="/var/lib/cni/networks/podnet"
mkdir -p $IPAM_STORE

# Allocate IP
function allocate_ip() {
  for i in {2..254}; do
    if [ ! -f "$IPAM_STORE/10.244.0.$i" ]; then
      echo "10.244.0.$i" > "$IPAM_STORE/10.244.0.$i"
      echo "10.244.0.$i"
      exit 0
    fi
  done
}

# Main logic
case $CNI_COMMAND in
  ADD)
    allocate_ip
    ;;
  DEL)
    rm -f "$IPAM_STORE/$POD_IP"
    ;;
esac
```

### Fixing Pod Connectivity and External Access in Kubernetes
```bash
# Enable IP forwarding
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

# Configure iptables for pod access
iptables -A FORWARD -s 10.244.0.0/16 -j ACCEPT
iptables -A FORWARD -d 10.244.0.0/16 -j ACCEPT
```

### Hands-On Lab: Network Policies

#### Basic Network Policy
```yaml
# deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### Allow Specific Traffic
```yaml
# allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 80
```

### Networking with Services

#### ClusterIP Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

#### NodePort Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

#### LoadBalancer Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-service
spec:
  type: LoadBalancer
  selector:
    app: public
  ports:
  - port: 80
    targetPort: 80
```

## Troubleshooting Guide

### Common Network Issues
1. Pod-to-Pod Communication
```bash
# Check pod connectivity
kubectl exec -it pod1 -- ping pod2

# Verify network policy
kubectl describe networkpolicy

# Check CNI configuration
cat /etc/cni/net.d/10-flannel.conf
```

2. Service Discovery Issues
```bash
# Test DNS resolution
kubectl exec -it pod1 -- nslookup kubernetes.default

# Check kube-dns/coredns
kubectl get pods -n kube-system
kubectl logs -n kube-system coredns-xxxxx
```

3. External Access Problems
```bash
# Verify node networking
ip route
iptables -L -t nat

# Check service external IP
kubectl get svc
```

### Monitoring and Debugging Tools
```bash
# Network traffic analysis
tcpdump -i any port 80

# Connection tracking
conntrack -L

# Service mesh debugging (if using Istio)
istioctl analyze
```