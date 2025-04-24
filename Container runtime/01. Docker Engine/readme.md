## 01. Installing a container runtime : All Nodes

### A) Set up Docker's apt repository.

**Official Documentation**: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### B) Install the Docker packages.

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### C) Verify that the Docker Engine installation is successful by running the hello-world image.

```bash
sudo docker run hello-world
```

### D) cri-docker install.
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker
- https://github.com/Mirantis/cri-dockerd

```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.17/cri-dockerd_0.3.17.3-0.debian-bullseye_amd64.deb
dpkg -i cri-dockerd_0.3.17.3-0.debian-bullseye_amd64.deb
systemctl status cri-docker
ls -l  /var/run/cri-dockerd.sock
```

<BR>

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

<BR>

## 03. Creating a cluster with kubeadm

### A) Initializing your control-plane node : k8s-master

```bash
# control-plane (master) component configuration
kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock


# To allow Kubectl to execute a command, you must act on the results of the execution of the kubadminit command.
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config


# Save the results of the execution of the kubeadm init command (kubeadm join command) separately.
kubeadm join 192.168.219.100:6443 --token 2whvdj...qbbib \
  --discovery-token-ca-cert-hash sha256:7125...78570b57 \
  --cri-socket unix:///var/run/cri-dockerd.sock 
```

### B) Installing a Pod network add-on

```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml


watch -n 3 kubectl get nodes
```

<BR>

## 04. Joining nodes - worker01, worker02

```bash
kubeadm join 192.168.219.100:6443 --token 2whv...bbib \
  --discovery-token-ca-cert-hash sha256:712538d8a5a...6a5078570b57 \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

<BR>

## 05. Default Configuration

### A) Set kubectl command autocomplete

```bash
# -> https://kubernetes.io/docs/reference/kubectl/quick-reference/
apt-get install bash-completion -y

source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### B) Install "etcd" - install ectcd on the control-plane (master)

```bash
export RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
wget https://github.com/etcd-io/etcd/releases/download/${RELEASE}/etcd-${RELEASE}-linux-amd64.tar.gz
tar xf etcd-${RELEASE}-linux-amd64.tar.gz
cd etcd-${RELEASE}-linux-amd64
mv etcd etcdctl etcdutl /usr/local/bin


etcd --version
```

<BR>

## 6. Final verification

```bash
$ kubectl get nodes -o wide
NAME            STATUS   ROLES           AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k8s-master      Ready    control-plane   2m14s   v1.32.4   192.168.219.10   <none>        Ubuntu 24.04.2 LTS   6.8.0-51-generic   docker://28.1.1
k8s-worker01    Ready    <none>          53s     v1.32.4   192.168.219.11   <none>        Ubuntu 24.04.2 LTS   6.8.0-51-generic   docker://28.1.1
k8s-worker02    Ready    <none>          44s     v1.32.4   192.168.219.12   <none>        Ubuntu 24.04.2 LTS   6.8.0-51-generic   docker://28.1.1
```

```bash
$ cat <<EOF > test_delpoy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.16
EOF

$ kubectl apply -f test_delpoy.yml

$ kubectl get all -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP          NODE            NOMINATED NODE   READINESS GATES
pod/web-deploy-585ff9c9b9-5dlwq   1/1     Running   0          13s   10.40.0.1   k8s-worker01    <none>           <none>
pod/web-deploy-585ff9c9b9-lfxh7   1/1     Running   0          13s   10.38.0.1   k8s-worker02    <none>           <none>
pod/web-deploy-585ff9c9b9-zxb68   1/1     Running   0          13s   10.38.0.2   k8s-worker02    <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   13m   <none>

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
deployment.apps/web-deploy   3/3     3            3           13s   nginx        nginx:1.16   app=web

NAME                                    DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES       SELECTOR
replicaset.apps/web-deploy-585ff9c9b9   3         3         3       13s   nginx        nginx:1.16   app=web,pod-template-hash=585ff9c9b9
```