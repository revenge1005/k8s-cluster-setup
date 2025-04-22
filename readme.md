# k8s Cluster Setup

## 01. Node Info

![node_info.drawio](https://github.com/revenge1005/k8s-cluster-setup/blob/main/node_info.drawio.png)

## 02. 사전 준비

- VMware Workstation : 가상 머신 생성, Ubuntu 24.04 설치
- Kubernetes 버전 : 1.32
- K8S 설치 공식 문서 : https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### A) k8s-master, k8s-worker1, k8s-worker2의 system 구성 설정 - All Node

```bash
cat <<EOF >>/etc/hosts

# Control Node.
192.168.219.100  k8s-master  
# Ansible Managed Nodes.
192.168.219.110  k8s-node01  
192.168.219.120  k8s-node02
EOF

# Swap disabled. 
{
	swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
	swapon -s
}
```

### B) IPv4를 포워딩하여 iptables가 bridge된 traffic을 보게 하기 - All Node

```bash
# Load "br_netfilter" module
{
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

	modprobe overlay
	modprobe br_netfilter
}

# Modifying "bridge traffic" kernel parameters
{
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
}
```

## 04. Choosing a container runtime.

- [1. Installing Kubernetes with Docker Engine Runtime]()
- [2. Installing Kubernetes with containerd Runtime]()