# k8s Cluster Setup

Kubernetes Cluster Setup Guide.

![node_info.drawio](https://github.com/revenge1005/k8s-cluster-setup/blob/main/node_info.drawio.png)

![k8s-arch](https://github.com/revenge1005/k8s-cluster-setup/blob/main/k8s-architecture.png)


## 01. 사전 준비

- **VMware Workstation** : Create virtual machines with Ubuntu 24.04.
- **Kubernetes Version** : 1.32
- **Official Documentation**: [Kubernetes Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### A) System Configuration for All Nodes (k8s-master, k8s-node01, k8s-node02)

```bash
# Add Kubernetes nodes to /etc/hosts for name resolution
cat <<EOF >>/etc/hosts

# Control Node.
192.168.219.100  k8s-master  
# Ansible Managed Nodes.
192.168.219.110  k8s-worker01  
192.168.219.120  k8s-worker02
EOF

# Disable swap to meet Kubernetes requirements
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
swapon -s
```

### B) Enable IPv4 Forwarding and Bridge Traffic - All Nodes

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

- [1. Installing Kubernetes with 【 Docker Engine (using cri-dockerd) Runtime 】](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)

  *Note*: Requires `cri-dockerd` for CRI compatibility in Kubernetes 1.32.

- [2. Installing Kubernetes with 【 containerd Runtime 】](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)


## 03. (Optional) Dynamic Volume Provisioning

- [1. 【 NFS 】](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20Dynamic%20Volume%20Provisioning/03-01.%20NFS)

- [2. 【 Ceph-csi 】](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20Dynamic%20Volume%20Provisioning/03-02.%20Ceph-csi)