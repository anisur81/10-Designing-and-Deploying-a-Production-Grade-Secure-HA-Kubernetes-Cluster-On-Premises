# Kubernetes High Availability Cluster Setup Guide

## 📋 Table of Contents
1. [Prerequisites](#prerequisites)
2. [Hardware Requirements](#hardware-requirements)
3. [Network Configuration](#network-configuration)
4. [Software Requirements](#software-requirements)
5. [Cluster Architecture](#cluster-architecture)
6. [Step-by-Step Installation](#step-by-step-installation)
7. [High Availability Configuration](#high-availability-configuration)
8. [Verification & Testing](#verification--testing)
9. [Troubleshooting](#troubleshooting)

---

## 1. Prerequisites

### System Requirements
- **8 Servers** running Ubuntu 26.04 LTS OS
- **Minimum Specifications per Node**:
  - RAM: 8 GB
  - CPU: 2 Cores
- **Root Password** configured on each server
- **Static IP Addresses** assigned to all nodes
- **Hostname Resolution** configured (DNS or /etc/hosts)
- **All nodes** must be reachable between each other

### Recommended Node Sizing

| Node Type | Hostname | Host IP | OS | Specs |
|-----------|----------|---------|-----|-------|
| **Master** | mstr1.aibl.com | 172.21.1.167 | Ubuntu 26.04 LTS | 8GB RAM, 2 vCPUs |
| **Master** | mstr2.aibl.com | 172.21.1.168 | Ubuntu 26.04 LTS | 8GB RAM, 2 vCPUs |
| **Master** | mstr3.aibl.com | 172.21.1.169 | Ubuntu 26.04 LTS | 8GB RAM, 2 vCPUs |
| **Worker** | wrk1.aibl.com | 172.21.1.170 | Ubuntu 26.04 LTS | 8GB RAM, 2 vCPUs |
| **Worker** | wrk2.aibl.com | 172.21.1.171 | Ubuntu 26.04 LTS | 8GB RAM, 2 vCPUs |
| **Worker** | wrk3.aibl.com | 172.21.1.172 | Ubuntu 26.04 LTS | 8GB RAM, 4 vCPUs |
| **Proxy** | HAProxy.aibl.com | 172.21.1.173 | Ubuntu 26.04 LTS | 8GB RAM, 4 vCPUs |
| **Storage** | Nfs.aibl.com | 172.21.1.174 | Ubuntu 26.04 LTS | 8GB RAM, 4 vCPUs |

---

## 2. Hardware Requirements

### Minimum vs Recommended

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **Master Node RAM** | 2 GB | 8+ GB |
| **Master Node CPU** | 2 cores | 4+ cores |
| **Worker Node RAM** | 2 GB | 8+ GB |
| **Worker Node CPU** | 2 cores | 4+ cores |
| **Storage (Master)** | 20 GB SSD | 50+ GB SSD |
| **Storage (Worker)** | 20 GB SSD | 100+ GB SSD |
| **Network** | 1 Gbps | 10 Gbps |

### Storage Requirements
- **etcd Database**: 10 GB minimum
- **Container Images**: 20 GB minimum
- **Application Data**: Varies based on workload
- **Logs**: 10 GB minimum

---

## 3. Network Configuration

### Network Topology

```
                     Users
                        |
                 Public Load Balancer
               (HAProxy / F5 / Nginx)
                        |
          +-------------------------------+
          |                               |
    Master Node 1                  Master Node 2
          |                               |
          +------------Master Node 3-------+
                    |
             etcd Cluster (HA)
                    |
      -------------------------------
      |             |              |
 Worker Node1  Worker Node2  Worker Node3
      |             |              |
      -------------------------------
                CNI Network
           (Calico / Cilium)
                    |
          Ingress Controller
           (NGINX / Traefik)
                    |
          Applications & Services
```

### IP Addressing Scheme

| Node | IP Address | Purpose |
|------|------------|---------|
| mstr1.aibl.com | 172.21.1.167 | Master Node 1 |
| mstr2.aibl.com | 172.21.1.168 | Master Node 2 |
| mstr3.aibl.com | 172.21.1.169 | Master Node 3 |
| wrk1.aibl.com | 172.21.1.170 | Worker Node 1 |
| wrk2.aibl.com | 172.21.1.171 | Worker Node 2 |
| wrk3.aibl.com | 172.21.1.172 | Worker Node 3 |
| HAProxy.aibl.com | 172.21.1.173 | Load Balancer |
| Nfs.aibl.com | 172.21.1.174 | NFS Storage |
| **VIP** | 172.21.1.175 | Virtual IP (HA) |
| **Service CIDR** | 10.96.0.0/12 | Kubernetes Services |
| **Pod CIDR** | 10.244.0.0/16 | Kubernetes Pods |

### Network Ports to Open

#### Master Nodes
| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API Server |
| 2379-2380 | TCP | etcd Server Client API |
| 10250 | TCP | Kubelet API |
| 10251 | TCP | kube-scheduler |
| 10252 | TCP | kube-controller-manager |
| 10259 | TCP | kube-scheduler (secure) |
| 10257 | TCP | kube-controller-manager (secure) |

#### Worker Nodes
| Port | Protocol | Purpose |
|------|----------|---------|
| 10250 | TCP | Kubelet API |
| 30000-32767 | TCP | NodePort Services |

#### All Nodes
| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH |
| 179 | TCP | Calico BGP |
| 4789 | UDP | VXLAN (Calico) |
| 5473 | TCP | Calico Typha |
| 9099 | TCP | Calico Felix |

### Network Configuration Steps

#### Step 1: Configure Static IP using Netplan

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    enp0s3:
      addresses:
        - 172.21.1.167/24
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      routes:
        - to: default
          via: 172.21.1.1
  version: 2
```

```bash
# Apply netplan configuration
sudo netplan apply

# Verify IP address
ip addr show enp0s3

# Verify default route
ip route show
```

#### Step 2: Configure Hostname Resolution

```bash
# On each node, update /etc/hosts
sudo cat >> /etc/hosts << EOF
172.21.1.167 mstr1.aibl.com mstr1
172.21.1.168 mstr2.aibl.com mstr2
172.21.1.169 mstr3.aibl.com mstr3
172.21.1.170 wrk1.aibl.com wrk1
172.21.1.171 wrk2.aibl.com wrk2
172.21.1.172 wrk3.aibl.com wrk3
172.21.1.173 haproxy.aibl.com haproxy
172.21.1.174 nfs.aibl.com nfs
172.21.1.175 vip.aibl.com vip
EOF

# Set hostname
sudo hostnamectl set-hostname mstr1.aibl.com
```

#### Step 3: Setup Passwordless SSH

```bash
# On master node (mstr1)
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa

# Copy public key to all nodes
for node in mstr1 mstr2 mstr3 wrk1 wrk2 wrk3 haproxy nfs; do
    ssh-copy-id root@$node.aibl.com
done

# Test connectivity
ssh root@wrk1.aibl.com
```

##Software Requirements
Ubuntu Server 22.04 LTS or 24.04 LTS

Static IP addresses assigned to all nodes

Hostname resolution configured (DNS or /etc/hosts)

All nodes must be reachable between each other

