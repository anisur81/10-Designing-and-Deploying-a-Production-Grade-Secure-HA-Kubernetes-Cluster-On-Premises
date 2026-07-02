# 🚀 Production-Grade HA Kubernetes Cluster Setup Guide

## 📋 Table of Contents
- [Network Prerequisites](#-network-prerequisites)
- [HAProxy Load Balancer Setup](#-haproxy-load-balancer-setup)
- [Node Configuration](#-node-configuration)
- [Container Runtime Installation](#-container-runtime-installation)
- [Kubernetes Components Installation](#-kubernetes-components-installation)
- [Control Plane Initialization](#-control-plane-initialization)
- [Joining Additional Nodes](#-joining-additional-nodes)
- [Network Plugin Installation](#-network-plugin-installation)
- [Verification & Testing](#-verification--testing)
- [Troubleshooting](#-troubleshooting)

---

## 🌐 Network Prerequisites

### 🔍 Verify Network Connectivity

**Ensure all nodes can communicate with each other:**

```bash
# From each node, test connectivity to all other nodes
ping -c 4 192.168.172.131  # master01
ping -c 4 192.168.172.132  # master02
ping -c 4 192.168.172.133  # master03
ping -c 4 192.168.172.146  # HAProxy VIP
```

### 📝 Update Hosts File (All Nodes)

```bash
cat <<EOF | sudo tee -a /etc/hosts
192.168.172.131 master01
192.168.172.132 master02
192.168.172.133 master03
192.168.172.134 worker01
192.168.172.135 worker02
192.168.172.136 worker03
192.168.172.146 haproxy-vip
EOF
```

### 🎯 Network Port Requirements

| Port Range | Purpose | Nodes |
|------------|---------|-------|
| 6443 | Kubernetes API Server | Control Plane |
| 2379-2380 | etcd server client API | Control Plane |
| 10250 | Kubelet API | All Nodes |
| 10259 | kube-scheduler | Control Plane |
| 10257 | kube-controller-manager | Control Plane |
| 8472 | Flannel/CNI | All Nodes |
| 30000-32767 | NodePort Services | All Nodes |

---

## ⚖️ HAProxy Load Balancer Setup

### 1️⃣ Install HAProxy

```bash
# Update package list and install HAProxy
sudo apt update && sudo apt install -y haproxy
```

### 2️⃣ Configure HAProxy

**Edit the configuration file:**

```bash
sudo vi /etc/haproxy/haproxy.cfg
```

**Add the following configuration:**

```haproxy
# Global Settings
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

# Default Settings
defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

# Frontend - Kubernetes API Server
frontend kubernetes
    bind 192.168.172.146:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

# Backend - Kubernetes Master Nodes
backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 192.168.172.131:6443 check fall 3 rise 2
    server master02 192.168.172.132:6443 check fall 3 rise 2
    server master03 192.168.172.133:6443 check fall 3 rise 2

# Optional: HAProxy Statistics Page
listen stats
    bind 192.168.172.146:9000
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats auth admin:secure_password_here
```

### 3️⃣ Start and Enable HAProxy

```bash
# Restart HAProxy to apply configuration
sudo systemctl restart haproxy

# Enable HAProxy to start on boot
sudo systemctl enable haproxy

# Verify HAProxy status
sudo systemctl status haproxy

# Check if HAProxy is listening on the VIP
sudo netstat -tulpn | grep 6443
# or
sudo ss -tulpn | grep 6443
```

---

## 🖥️ Node Configuration (All Nodes)

### 1️⃣ Disable Swap

```bash
# Disable swap immediately
sudo swapoff -a

# Comment out swap entry in /etc/fstab
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**Verify swap is disabled:**
```bash
free -m
# Swap should show 0 total
```

### 2️⃣ Load Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load the modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Verify modules are loaded
lsmod | grep br_netfilter
lsmod | grep overlay
```

### 3️⃣ Configure Sysctl for Kubernetes Networking

```bash
# Create Kubernetes sysctl configuration
sudo vi /etc/sysctl.d/kubernetes.conf
```

**Add these lines:**
```ini
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

**Apply sysctl settings:**
```bash
sudo sysctl --system
```

**Verify settings:**
```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

### 4️⃣ Install Required Packages

```bash
# Update package list
sudo apt-get update

# Install prerequisites
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common \
    net-tools \
    vim
```

---

## 📦 Container Runtime Installation (All Nodes)

### Install Containerd

```bash
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Create default configuration directory
sudo mkdir -p /etc/containerd

# Generate default configuration
containerd config default | sudo tee /etc/containerd/config.toml

# Configure systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo systemctl status containerd
```

---

## 🚀 Kubernetes Components Installation (All Nodes)

### 1️⃣ Add Kubernetes Repository

```bash
# Download Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
    https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
    sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list
sudo apt-get update
```

### 2️⃣ Install Kubernetes Components

```bash
# Install kubeadm, kubelet, and kubectl
sudo apt-get install -y kubelet kubeadm kubectl

# Hold packages to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3️⃣ Configure kubelet

```bash
# Set kubelet cgroup driver
echo "KUBELET_EXTRA_ARGS=--cgroup-driver=systemd" | \
    sudo tee /etc/default/kubelet

# Reload systemd and restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---

## 🎯 Control Plane Initialization

### 🔴 Step 1: Initialize First Master Node (master01)

```bash
# On master01 (192.168.172.131)
sudo kubeadm init \
    --control-plane-endpoint="192.168.172.146:6443" \
    --upload-certs \
    --apiserver-advertise-address=192.168.172.131 \
    --pod-network-cidr=192.168.0.0/16 \
    --ignore-preflight-errors all
```

**Alternative initialization commands:**

```bash
# If using different network CIDR
sudo kubeadm init \
    --control-plane-endpoint="192.168.172.146:6443" \
    --upload-certs \
    --apiserver-advertise-address=172.21.1.167 \
    --pod-network-cidr=192.168.0.0/16

# Or with default settings
sudo kubeadm init \
    --control-plane-endpoint="192.168.172.146:6443" \
    --upload-certs
```

### 📝 Save Important Output

**The `kubeadm init` command will output:**

1. **Control Plane Join Command:**
```bash
kubeadm join 192.168.172.146:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <key>
```

2. **Worker Node Join Command:**
```bash
kubeadm join 192.168.172.146:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### 📂 Configure kubectl for First Node

```bash
# Create .kube directory
mkdir -p $HOME/.kube

# Copy admin configuration
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Set ownership
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify kubectl access
kubectl get nodes
kubectl get pods -n kube-system
```

### 🔄 If You Lose the Join Commands

**Generate new token for worker nodes:**
```bash
kubeadm token create --print-join-command
```

**Generate new certificate key for control plane nodes:**
```bash
kubeadm init phase upload-certs --upload-certs
```

---

## 🔗 Joining Additional Nodes

### 🟠 Step 2: Join Additional Control Plane Nodes

**On master02 (192.168.172.132):**
```bash
sudo kubeadm join 192.168.172.146:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <key> \
    --apiserver-advertise-address=192.168.172.132
```

**On master03 (192.168.172.133):**
```bash
sudo kubeadm join 192.168.172.146:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <key> \
    --apiserver-advertise-address=192.168.172.133
```

### 🟢 Step 3: Join Worker Nodes

**On worker01 (192.168.172.134):**
```bash
sudo kubeadm join 192.168.172.146:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

**On worker02 (192.168.172.135):**
```bash
sudo kubeadm join 192.168.172.146:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

**On worker03 (192.168.172.136):**
```bash
sudo kubeadm join 192.168.172.146:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### ✅ Verify Cluster Nodes

**From any control plane node:**
```bash
kubectl get nodes -o wide

# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP      EXTERNAL-IP
# master01   Ready    control-plane   10m   v1.29.x   192.168.172.131  <none>
# master02   Ready    control-plane   8m    v1.29.x   192.168.172.132  <none>
# master03   Ready    control-plane   6m    v1.29.x   192.168.172.133  <none>
# worker01   Ready    <none>          4m    v1.29.x   192.168.172.134  <none>
# worker02   Ready    <none>          3m    v1.29.x   192.168.172.135  <none>
# worker03   Ready    <none>          2m    v1.29.x   192.168.172.136  <none>
```

---

## 🌐 Network Plugin Installation

### Option 1: Flannel (Recommended)

```bash
# Install Flannel CNI
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Verify Flannel pods
kubectl get pods -n kube-system -l app=flannel -o wide
```

### Option 2: Calico (Advanced Features)

```bash
# Install Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27/manifests/calico.yaml

# Verify Calico pods
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
```

### Option 3: Cilium (eBPF-based)

```bash
# Install Cilium using Helm
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=disabled

# Verify Cilium pods
kubectl get pods -n kube-system -l k8s-app=cilium -o wide
```

---

## ✅ Verification & Testing

### 1️⃣ Cluster Health Check

```bash
# Check all nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check cluster info
kubectl cluster-info

# Check component status
kubectl get componentstatuses
```

### 2️⃣ Network Connectivity Test

```bash
# Deploy test pod
kubectl run test-pod --image=nginx --restart=Never --port=80

# Get pod details
kubectl get pods -o wide

# Test connectivity to pod
kubectl exec -it test-pod -- sh

# Inside the pod:
apt-get update && apt-get install -y curl
curl -I 192.168.0.1  # Test CNI network
exit

# Create service
kubectl expose pod test-pod --type=ClusterIP --port=80 --name=test-service

# Test service discovery
kubectl exec -it test-pod -- sh -c "apt-get update && apt-get install -y curl && curl -I test-service"
```

### 3️⃣ HAProxy Load Balancer Test

```bash
# Test API endpoint directly
curl -k https://192.168.172.146:6443/healthz

# Check HAProxy stats
curl http://192.168.172.146:9000/stats

# Check HAProxy logs
sudo tail -f /var/log/haproxy.log
```

### 4️⃣ Control Plane HA Test

```bash
# On one control plane node, stop kubelet
sudo systemctl stop kubelet

# Check cluster status - API should still be accessible
kubectl get nodes

# Restart kubelet
sudo systemctl start kubelet

# Verify nodes become Ready again
kubectl get nodes -w
```

### 5️⃣ DNS Test

```bash
# Test DNS resolution
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
    nslookup kubernetes.default.svc.cluster.local
```

### 6️⃣ Storage Test (If NFS Configured)

```yaml
# test-storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
```

```bash
# Apply and test
kubectl apply -f test-storage.yaml
kubectl get pv,pvc
kubectl delete -f test-storage.yaml
```

---

## 🔧 Troubleshooting

### Issue 1: Node Not Ready

```bash
# Check node conditions
kubectl describe node <node-name>

# Check kubelet logs
sudo journalctl -u kubelet -f

# Check node status
kubectl get node <node-name> -o yaml

# Restart kubelet
sudo systemctl restart kubelet
```

### Issue 2: CNI Problems

```bash
# Check CNI pods
kubectl get pods -n kube-system | grep -E 'flannel|calico|cilium'

# Check CNI configuration
ls -la /etc/cni/net.d/

# View CNI config
cat /etc/cni/net.d/10-flannel.conflist
```

### Issue 3: etcd Issues

```bash
# Check etcd health
kubectl exec -n kube-system etcd-master01 -- etcdctl endpoint health

# Check etcd logs
kubectl logs -n kube-system etcd-master01

# Check etcd members
kubectl exec -n kube-system etcd-master01 -- etcdctl member list
```

### Issue 4: API Server Connectivity

```bash
# Test API endpoint
curl -k https://192.168.172.146:6443/healthz

# Check HAProxy configuration
sudo cat /etc/haproxy/haproxy.cfg

# Test HAProxy stats
curl http://192.168.172.146:9000/stats

# Check kubeconfig
kubectl cluster-info --v=10
```

### Issue 5: Pod Networking

```bash
# Check network policies
kubectl get networkpolicies --all-namespaces

# Test pod-to-pod connectivity
kubectl run test-pod1 --image=nginx --restart=Never
kubectl run test-pod2 --image=busybox --restart=Never -- sleep 3600

# Get pod IPs
kubectl get pods -o wide

# Test connectivity
kubectl exec -it test-pod2 -- sh -c "ping <test-pod1-ip>"
```

### Issue 6: Certificate Issues

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Renew certificates
sudo kubeadm certs renew all

# Restart API server
sudo systemctl restart kubelet
```

---

## 📊 Useful Commands Reference

### Quick Status Checks

```bash
# Complete cluster status
kubectl get nodes,pods,services,deployments,statefulsets --all-namespaces

# Resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# System pods
kubectl get pods -n kube-system -o wide | grep -v Running
```

### Backup Commands

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db

# Backup all resources
kubectl get all --all-namespaces -o yaml > /backup/k8s-all-backup.yaml

# Backup specific resources
kubectl get secrets --all-namespaces -o yaml > /backup/secrets-backup.yaml
kubectl get configmaps --all-namespaces -o yaml > /backup/configmaps-backup.yaml
```

---

## 📝 Summary Checklist

- [ ] All nodes have static IPs and are reachable
- [ ] `/etc/hosts` updated on all nodes
- [ ] Swap disabled on all nodes
- [ ] Kernel modules loaded
- [ ] Sysctl settings applied
- [ ] Containerd installed and configured
- [ ] Kubernetes components installed
- [ ] HAProxy installed and configured
- [ ] First control plane initialized
- [ ] Additional control plane nodes joined
- [ ] Worker nodes joined
- [ ] Network plugin installed
- [ ] All nodes are Ready
- [ ] kubectl works from control plane
- [ ] Cluster HA tested

---

## 🔗 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubeadm Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [HAProxy Documentation](https://www.haproxy.org/#docs)
- [Flannel Documentation](https://github.com/flannel-io/flannel#readme)
- [Calico Documentation](https://docs.tigera.io/calico/latest/)
- [Cilium Documentation](https://docs.cilium.io/)

---

**🎉 Congratulations!** Your production-grade HA Kubernetes cluster is now ready!
