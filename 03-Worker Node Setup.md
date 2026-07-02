# Kubernetes Worker Node Setup Guide

## Complete Setup Process for Worker Node

This document outlines the complete setup process for a Kubernetes worker node based on the command history provided.

---

## 1. Initial System Configuration

### Set Hostname
```bash
hostnamectl set-hostname worker01
```

### Update Hosts File
```bash
cat /etc/hosts
# Ensure proper hostname resolution
```

---

## 2. Docker Installation

### Install Prerequisites
```bash
apt install apt-transport-https ca-certificates curl software-properties-common -y
```

### Add Docker GPG Key
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### Add Docker Repository
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker
```bash
apt update -y
apt install docker-ce -y
systemctl enable docker --now
docker --version
```

---

## 3. Kubernetes Installation

### Add Kubernetes Repository
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Kubernetes Components
```bash
apt update -y
apt install kubeadm kubelet -y
```

---

## 4. System Preparation

### Disable Swap (Required for Kubernetes)
```bash
swapoff -a
# Edit /etc/fstab and remove/comment the swap line
vi /etc/fstab
```

### Load Required Kernel Modules
```bash
vi /etc/modules-load.d/k8s.conf
```
Add the following content:
```
overlay
br_netfilter
```

Load modules:
```bash
modprobe overlay
modprobe br_netfilter
```

### Configure Sysctl Parameters
```bash
vi /etc/sysctl.d/10-k8s.conf
```
Add the following content:
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

Apply sysctl settings:
```bash
sysctl --system
```

---

## 5. Kubelet Configuration

### Configure Cgroup Driver
```bash
vi /etc/default/kubelet
```
Add:
```
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
```

### Restart Services
```bash
systemctl daemon-reload
systemctl restart kubelet.service
```

---

## 6. Containerd Configuration

### Reset Containerd Configuration
```bash
rm -rf /etc/containerd/config.toml
systemctl restart containerd.service
```

---

## 7. Join the Cluster

### Join Command Examples
```bash
# Example 1
kubeadm join 172.200.211.104:6443 --token v73r8y.kb435vyius9i0jp7 --discovery-token-ca-cert-hash sha256:dc2f22d55d09f23dc46c809cfc81801805cb31f95b3ac9ee300fb331f90420fb

# Example 2
kubeadm join 192.168.233.130:6443 --token n2fns7.r57ndu80jmtkgb32 --discovery-token-ca-cert-hash sha256:9c19ef7570d7ab448f2fa969228d0c4f9c15b3abd1f33e4929cfb18d74452580

# Example 3
kubeadm join 172.21.1.173:6443 --token x1dmh5.w0nnknskd3d5s0vy --discovery-token-ca-cert-hash sha256:2bbd80eb3549cccb6c2f944416cf092db291538e7c883f51dcc5ca530b011498
```

---

## 8. Verification

### Check Container Images
```bash
crictl image ls
```

---

## Troubleshooting

### Common Issues

1. **Swap still enabled**: Ensure `swapoff -a` and remove swap line from `/etc/fstab`
2. **Kernel modules not loaded**: Verify with `lsmod | grep br_netfilter`
3. **Cgroup driver mismatch**: Ensure kubelet and containerd use same cgroup driver
4. **Token expired**: Generate new token on master with `kubeadm token create`
5. **Network issues**: Check connectivity to master node on port 6443

### Useful Commands

```bash
# Check kubelet status
systemctl status kubelet

# Check containerd status
systemctl status containerd

# View kubelet logs
journalctl -u kubelet -f

# View containerd logs
journalctl -u containerd -f
```

---

**Note**: This documentation is based on the command history provided and should be adapted based on your specific environment and requirements.
