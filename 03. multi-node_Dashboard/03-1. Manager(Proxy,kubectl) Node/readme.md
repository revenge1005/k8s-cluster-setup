# 0. 선행 작업

- [**02-1. Installing a container runtime (containerd) : k8s-master, k8s-worker01, k8s-worker02 node**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd#01-installing-a-container-runtime-containerd--all-nodes)

- [**02-2. Installing kubeadm : k8s-master, k8s-worker01, k8s-worker02 node**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd#02-installing-kubeadm--all-node)

<BR>

# 1. Configure Manager(Proxy,kubectl) Node

![multi-node](https://github.com/revenge1005/k8s-cluster-setup/blob/main/multi-node-configuration.png)

<BR>

### A) Configure Manager Node first. : k8s-mgs

```bash
apt -y install nginx libnginx-mod-stream
```

```bash
$ vi /etc/nginx/nginx.conf

# add to last line : proxy settings
stream {
    upstream k8s-api {
        server 192.168.219.100:6443;
    }
    server {
        listen 6443;
        proxy_pass k8s-api;
    }
}
```

```bash
# disable default site on Nginx
unlink /etc/nginx/sites-enabled/default
systemctl restart nginx
```

<BR>

### B) On Manager Node, Install Kubernetes client. : k8s-mgs

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt update && apt -y install kubectl
```

<BR>

### C) Configure initial setup on Control Plane Node. : k8s-master

```bash
kubeadm init --control-plane-endpoint=192.168.219.25 \
	--apiserver-advertise-address=192.168.219.100 \
	--pod-network-cidr=192.168.0.0/16 \
	--cri-socket=unix:///run/containerd/containerd.sock
```

```bash
# transfer authentication file for cluster admin to Manager Node with any user
scp /etc/kubernetes/admin.conf root@192.168.219.25:/tmp
```

### D) Work on Manager Node. : k8s-mgs

```bash
# set cluster admin user with a file you transferred from Control Plane
# if you set common user as cluster admin, login with it and run [sudo cp/chown ***]
mkdir -p $HOME/.kube
mv /tmp/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
root@k8s-mgs:~# kubectl get nodes -o wide
NAME         STATUS     ROLES           AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k8s-master   NotReady   control-plane   60s   v1.32.4   192.168.219.100   <none>        Ubuntu 24.04.2 LTS   6.8.0-60-generic   containerd://1.7.24
```

### E) Installing a Pod network add-on - k8s-mgs

```bash
{
    kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
    watch -n 3 kubectl get nodes
}
```

### F) Set kubectl command autocomplete - k8s-mgs

```bash
{
    source <(kubectl completion bash)
    echo "source <(kubectl completion bash)" >> ~/.bashrc
}
```

### G) Install "etcd" : install ectcd on the control-plane - k8s-master

```bash
export RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
wget https://github.com/etcd-io/etcd/releases/download/${RELEASE}/etcd-${RELEASE}-linux-amd64.tar.gz
tar xf etcd-${RELEASE}-linux-amd64.tar.gz
cd etcd-${RELEASE}-linux-amd64
mv etcd etcdctl etcdutl /usr/local/bin


etcd --version
```

### H) Start the next step.

- [03-2. Add Control Plane Node](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-2.%20Add%20Control%20Plane%20Node)

- [03-3. Add Worker Node](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-3.%20Add%20Worker%20Node)