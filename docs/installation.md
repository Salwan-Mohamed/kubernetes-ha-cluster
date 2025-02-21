# Kubernetes HA Cluster Installation Guide

This guide provides detailed instructions for setting up a high-availability Kubernetes cluster with external etcd.

## Prerequisites

- 11 Ubuntu 24.04 LTS servers with the following minimum specifications:
  - 2 vCPUs
  - 4GB RAM
  - 50GB disk space
  - Static IP addresses configured
- SSH access to all servers
- Internet connectivity for package installation
- Hostnames configured according to the architecture

## Network Architecture

| Purpose | Hostname | IP Address |
|---------|----------|------------|
| Control Plane | master01 | 10.1.5.2 |
| Control Plane | master02 | 10.1.5.3 |
| Control Plane | master03 | 10.1.5.4 |
| etcd | etcd01 | 10.1.5.5 |
| etcd | etcd02 | 10.1.5.6 |
| etcd | etcd03 | 10.1.5.7 |
| Load Balancer | haproxy01 | 10.1.5.8 |
| Load Balancer | haproxy02 | 10.1.5.9 |
| Virtual IP | k8s-vip | 10.1.5.10 |
| Worker | worker01 | 10.1.5.11 |
| Worker | worker02 | 10.1.5.12 |
| Worker | worker03 | 10.1.5.13 |

## Step 1: Configure All Nodes

Perform these steps on all 11 servers.

### 1.1 Configure /etc/hosts

```bash
cat << EOF | sudo tee -a /etc/hosts
10.1.5.2 master01
10.1.5.3 master02
10.1.5.4 master03
10.1.5.5 etcd01
10.1.5.6 etcd02
10.1.5.7 etcd03
10.1.5.8 haproxy01
10.1.5.9 haproxy02
10.1.5.10 k8s-vip
10.1.5.11 worker01
10.1.5.12 worker02
10.1.5.13 worker03
EOF
```

### 1.2 Update System and Install Prerequisites

```bash
sudo apt update && sudo apt upgrade -y

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure kernel parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 1.3 Install Container Runtime (containerd)

```bash
# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Install containerd
sudo apt install -y containerd

# Configure containerd to use systemd cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Edit config to use systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 1.4 Install Kubernetes Packages (Skip on etcd-only nodes)

Skip this step on etcd01, etcd02, and etcd03.

```bash
# Add Kubernetes apt repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubeadm, kubelet and kubectl
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Pin their version to avoid unexpected upgrades
sudo apt-mark hold kubelet kubeadm kubectl
```

## Step 2: Configure Load Balancers

### 2.1 Install HAProxy (on haproxy01 and haproxy02)

```bash
sudo apt install -y haproxy
```

### 2.2 Configure HAProxy

Create the HAProxy configuration file on both load balancer nodes:

```bash
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend kubernetes-frontend
    bind *:6443
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 10.1.5.2:6443 check fall 3 rise 2
    server master02 10.1.5.3:6443 check fall 3 rise 2
    server master03 10.1.5.4:6443 check fall 3 rise 2
EOF

sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

### 2.3 Install and Configure Keepalived

#### On haproxy01 (Primary)

```bash
sudo apt install -y keepalived

cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens3
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass K8SHA_KP
    }
    virtual_ipaddress {
        10.1.5.10
    }
    track_script {
        check_haproxy
    }
}
EOF

sudo systemctl restart keepalived
sudo systemctl enable keepalived
```

#### On haproxy02 (Backup)

```bash
sudo apt install -y keepalived

cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens3
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass K8SHA_KP
    }
    virtual_ipaddress {
        10.1.5.10
    }
    track_script {
        check_haproxy
    }
}
EOF

sudo systemctl restart keepalived
sudo systemctl enable keepalived
```

### 2.4 Verify Virtual IP

On haproxy01, verify that the virtual IP has been assigned:

```bash
ip addr show ens3
```

You should see the 10.1.5.10 address listed. If you stop the keepalived service on haproxy01, the IP should move to haproxy02.

## Step 3: Configure External etcd Cluster

### 3.1 Install etcd on Dedicated Nodes

Run these commands on etcd01, etcd02, and etcd03:

```bash
# Download etcd binary
ETCD_VER=v3.5.9
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz

# Move binaries to system path
sudo mv etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/
sudo mv etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
sudo mv etcd-${ETCD_VER}-linux-amd64/etcdutl /usr/local/bin/

# Create necessary directories
sudo mkdir -p /var/lib/etcd
sudo mkdir -p /etc/etcd
```

### 3.2 Configure etcd on Each Node

#### On etcd01

```bash
cat <<EOF | sudo tee /etc/default/etcd
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_CLIENT_URLS="http://10.1.5.5:2379,http://127.0.0.1:2379"
ETCD_LISTEN_PEER_URLS="http://10.1.5.5:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.1.5.5:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.1.5.5:2380"
ETCD_INITIAL_CLUSTER="etcd01=http://10.1.5.5:2380,etcd02=http://10.1.5.6:2380,etcd03=http://10.1.5.7:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
EOF

# Create systemd service file
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
Type=notify
EnvironmentFile=/etc/default/etcd
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start etcd
sudo systemctl enable etcd
```

#### On etcd02

```bash
cat <<EOF | sudo tee /etc/default/etcd
ETCD_NAME="etcd02"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_CLIENT_URLS="http://10.1.5.6:2379,http://127.0.0.1:2379"
ETCD_LISTEN_PEER_URLS="http://10.1.5.6:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.1.5.6:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.1.5.6:2380"
ETCD_INITIAL_CLUSTER="etcd01=http://10.1.5.5:2380,etcd02=http://10.1.5.6:2380,etcd03=http://10.1.5.7:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
EOF

# Create systemd service file
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
Type=notify
EnvironmentFile=/etc/default/etcd
ExecStart=/usr/local/bin/etcd
