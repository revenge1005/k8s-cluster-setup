# 0. 선행 작업

- [1. Installing a container runtime (containerd)](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd#01-installing-a-container-runtime-containerd--all-nodes)

- [2. Installing kubeadm](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd#02-installing-kubeadm--all-node)

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


# Save the results of the execution of the kubeadm init command (kubeadm join command) separately.
kubeadm join 192.168.219.25:6443 --token 2whvdj...qbbib \
  --discovery-token-ca-cert-hash sha256:7125...78570b57 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

```bash
root@k8s-mgs:~# kubectl get nodes -o wide
NAME         STATUS     ROLES           AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k8s-master   NotReady   control-plane   60s   v1.32.4   192.168.219.100   <none>        Ubuntu 24.04.2 LTS   6.8.0-60-generic   containerd://1.7.24
```

### E) Start the next step.

- [3-B. Installing a Pod network add-on - k8s-master](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd#b-installing-a-pod-network-add-on---k8s-master)