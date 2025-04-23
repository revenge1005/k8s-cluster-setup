## 01. Installing a container runtime : All Nodes

```bash
apt -y install containerd
mkdir /etc/containerd
containerd config default | tee /etc/containerd/config.toml


vi /etc/containerd/config.toml
# line 65 : change
sandbox_image = "registry.k8s.io/pause:3.9"

# line 137 : change
SystemdCgroup = true

systemctl restart containerd.service
```

## 02. Installing kubeadm : ALL Node

### A) Installing kubeadm, kubelet and kubectl

- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

```bash
# Update the apt package index and install packages needed to use the Kubernetes apt repository:
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg


# Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg


# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


# Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


# (Optional) Enable the kubelet service before running kubeadm:
sudo systemctl enable --now kubelet
```

## 03. Creating a cluster with kubeadm

### A) Initializing your control-plane node : k8s-master

```bash
# control-plane (master) component configuration
kubeadm init --pod-network-cidr=192.168.0.0/16 \
    --cri-socket=unix:///run/containerd/containerd.sock

# To allow Kubectl to execute a command, you must act on the results of the execution of the kubadminit command.
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config


# Save the results of the execution of the kubeadm init command (kubeadm join command) separately.
kubeadm join 192.168.219.10:6443 --token 2whvdj...qbbib \
  --discovery-token-ca-cert-hash sha256:7125...78570b57 \
  --cri-socket unix:///var/run/cri-dockerd.sock 
```

### B) Installing a Pod network add-on

```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml


watch -n 3 kubectl get nodes
```

## 04. Joining nodes - worker01, worker02

```bash
kubeadm join 192.168.219.10:6443 --token 2whv...bbib \
  --discovery-token-ca-cert-hash sha256:712538d8a5a...6a5078570b57 \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

## 05. Default Configuration

### A) Set kubectl command autocomplete

```bash
# -> https://kubernetes.io/docs/reference/kubectl/quick-reference/
apt-get install bash-completion -y

source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
