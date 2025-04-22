# k8s Cluster Setup

Kubernetes Cluster Setup Guide.

![node_info.drawio](https://github.com/revenge1005/k8s-cluster-setup/blob/main/node_info.drawio.png)

## 01. 사전 준비 - All Node

- **VMware Workstation** : Create virtual machines with Ubuntu 24.04.
- **Kubernetes Version** : 1.32
- **Official Documentation**: [Kubernetes Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### A) k8s-master, k8s-worker1, k8s-worker2의 system 구성 설정

```bash
cat <<EOF >>/etc/hosts

# Control Node.
192.168.219.100  k8s-master  
# Ansible Managed Nodes.
192.168.219.110  k8s-node01  
192.168.219.120  k8s-node02
EOF

# Swap disabled. 
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
swapon -s
```

### B) IPv4를 포워딩하여 iptables가 bridge된 traffic을 보게 하기

```bash
# Load required kernel modules for container networking
{
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

	modprobe overlay
	modprobe br_netfilter
}

# Configure sysctl for bridge traffic and IPv4 forwarding
{
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
}
```

## 02. Choosing a container runtime.

- [1. Installing Kubernetes with Docker Engine Runtime](https://github.com/revenge1005/k8s-cluster-setup/blob/main/01.%20Docker%20Engine/readme.md)

  *Note*: Requires `cri-dockerd` for CRI compatibility in Kubernetes 1.32.

- [2. Installing Kubernetes with containerd Runtime](https://github.com/revenge1005/k8s-cluster-setup/blob/main/02.%20containerd/readme.md)