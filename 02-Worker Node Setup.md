# Kubernetes Cluster Setup with Calico Networking

## Network Configuration

### 1. Netplan Configuration
```bash
# Navigate to netplan directory
cd /etc/netplan/

# Edit network configuration
vi 50-cloud-init.yaml
```

```yaml
network:
    ethernets:
        ens192:
            dhcp4: false
            addresses: [172.200.211.104/24]
            gateway4: 172.200.211.1
            nameservers:
             addresses: [8.8.8.8,12.168.100.10]
    version: 2
```

```bash
# Apply network configuration
netplan apply
```

## Docker Installation

### Install Docker Dependencies
```bash
apt install apt-transport-https ca-certificates curl software-properties-common -y
```

### Add Docker GPG Key and Repository
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install and Start Docker
```bash
apt update -y
apt install docker-ce -y
systemctl enable docker --now
docker --version
```

## Host Configuration

### Update Hosts File
```bash
cat /etc/hosts
```
```
172.200.211.104 master1
172.200.211.105 worker1
172.200.211.106 worker2
```

### Set Hostname
```bash
hostnamectl set-hostname master1
```

## Kubernetes Installation

### Add Kubernetes Repository
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Kubernetes Components
```bash
apt update -y
apt install kubeadm kubelet kubectl -y
swapoff -a
```

### Verify Installation
```bash
kubeadm version
kubelet --version
kubectl version
```

## Kernel Module Configuration

### Load Required Modules
```bash
vi /etc/modules-load.d/k8s.conf
```
```
modprobe overlay
modprobe br_netfilter
```

```bash
modprobe overlay
modprobe br_netfilter
lsmod | grep over
lsmod | grep br_netfilter
```

### Configure Sysctl Parameters
```bash
vi /etc/sysctl.d/10-k8s.conf
```
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

```bash
sysctl --system
```

## Kubelet Configuration

```bash
cat /etc/default/kubelet
```
```
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
```

### Restart Services
```bash
systemctl daemon-reload
systemctl restart kubelet.service
```

## Container Runtime Configuration

```bash
rm -rf /etc/containerd/config.toml
systemctl restart containerd.service
systemctl status containerd.service
```

## Kubernetes Cluster Initialization

### Pull Required Images
```bash
kubeadm config images pull
```

### Initialize the Cluster
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

#### Alternative Init Commands
```bash
# With specific advertise address
sudo kubeadm init --apiserver-advertise-address=192.168.172.131 --pod-network-cidr=10.244.0.0/16

# For multi-master cluster
kubeadm init --control-plane-endpoint="172.21.1.173:6443" --upload-certs --apiserver-advertise-address=172.21.1.167 --pod-network-cidr=192.168.0.0/16
```

## Configure kubectl

### Setup Kubeconfig
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Enable Bash Completion
```bash
vi .bashrc
```
```
export KUBECONFIG=/etc/kubernetes/admin.conf
source <(kubectl completion bash)
```

```bash
source .bashrc
apt-get install bash-completion -y
```

## Calico Networking Installation

### Create Working Directory
```bash
mkdir calico
cd calico/
```

### Download Calico Manifests
```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
```

### Configure Calico Network
```bash
cat custom-resources.yaml
```

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

### Apply Calico Manifests
```bash
# Apply operator manifest
kubectl create -f tigera-operator.yaml

# Apply custom resources
kubectl create -f custom-resources.yaml
```

### Alternative Calico Installation
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml
```

## Verify Cluster Status

### Check Resources
```bash
kubectl get nodes
kubectl get pods -A
kubectl describe nodes master1
```

### Health Check
```bash
kubectl get --raw='/readyz?verbose'
curl -k https://localhost:6443/livez?verbose
```

## Joining Nodes to Cluster

### Generate Join Command
```bash
kubeadm token create --print-join-command
```

### Join Control Plane Nodes (Multi-Master)
```bash
kubeadm join 172.21.1.173:6443 --token x1dmh5.w0nnknskd3d5s0vy \
        --discovery-token-ca-cert-hash sha256:2bbd80eb3549cccb6c2f944416cf092db291538e7c883f51dcc5ca530b011498 \
        --control-plane --certificate-key 491f2ef94a31a12dcef302e9ebb38e4b2dd906e25cfce5148b017cb8213ec9cb
```

---

## Troubleshooting Commands

- **Check Docker status:** `systemctl status docker`
- **Check Kubernetes services:** `systemctl status kubelet`
- **Check Calico pods:** `kubectl get pods -n calico-system`
- **View cluster events:** `kubectl get events --all-namespaces`
- **Check node conditions:** `kubectl describe node <node-name>`
