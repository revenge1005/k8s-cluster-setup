# k8s Cluster Setup

![node_info.drawio](https://github.com/revenge1005/k8s-cluster-setup/blob/main/node_info.drawio.png)

## Installation and Running

### 1. 설치 환경

- **VMware Workstation** : Create virtual machines with Ubuntu 24.04.
- **Kubernetes Version** : 1.32
- **Official Documentation**: [Kubernetes Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- **CPU / Memory Setting(k8s-master,k8s-worker01,k8s-worker02)** : 

![cpu_memory](https://github.com/revenge1005/k8s-cluster-setup/blob/main/k8s_cpu_memory.PNG)

### 2. 사전 준비

#### A) System Configuration for All Nodes 

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

#### B) Enable IPv4 Forwarding and Bridge Traffic - All Nodes

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

### 3. Container runtime (Select only one and install it.)

* [**Installing Kubernetes with 【 containerd Runtime 】**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)

* [**Installing Kubernetes with 【 Docker Engine (using cri-dockerd) Runtime 】**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)
  * *Note*: Requires `cri-dockerd` for CRI compatibility in Kubernetes 1.32.

### 4. (Optional) K8s Multi Node / Dashboard

* [**A) Manager(Proxy,kubectl) Node**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-1.%20Manager(Proxy%2Ckubectl)%20Node)

* [**B) Add Control Plane Node**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-2.%20Add%20Control%20Plane%20Node)

* [**C) Add Worker Node**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-3.%20Add%20Worker%20Node)

* [**D) Enable Dashboard**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-4.%20Enable%20Dashboard)

* [**E) Remove Nodes**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-5.%20Remove%20Nodes)

### 5. (Optional) Set up a LoadBalancer for bare-metal clusters using MetalLB.

* [**A) Installing MetalLB**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/04.%20MetalLB)

### 6. (Optional) Dynamic Volume Provisioning (Select only one and install it.)

* [**A) NFS**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/05.%20Dynamic%20Volume%20Provisioning/05-01.%20NFS)

* [**B) Ceph-csi (Cephfs)**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/05.%20Dynamic%20Volume%20Provisioning/05-02.%20Ceph-csi(cephfs))

* [**C) Ceph-csi (RBD)**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/05.%20Dynamic%20Volume%20Provisioning/05-03.%20Ceph-csi(rbd))

* [**D) Rook ceph**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/05.%20Dynamic%20Volume%20Provisioning/05-04.%20rook_ceph)

### 7. Ingress Controller

* [**A) Ingress-Nginx Controller**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/06.%20Ingress-Nginx%20Controller)

### 8. Monitoring

* [**A) Deploy Prometheus, Grafana, Loki**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/07.%20Monitoring/07-1.%20config)

* [**B) Example Prometheus Dashboard**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/07.%20Monitoring/07-2.%20example)