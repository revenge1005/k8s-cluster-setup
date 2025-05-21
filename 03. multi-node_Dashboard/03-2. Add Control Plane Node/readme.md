# 0. 선행 작업

- [Manager(Proxy,kubectl) Node](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)

<br>

# 1. Add Control Plane Node

![multi-node](https://github.com/revenge1005/k8s-cluster-setup/blob/main/multi-node-configuration.png)

<BR>

### A) Confirm join command on existing Control Plane Node and also transfer certificate files to new Node with any user.

```bash
# k8s-master-1 node

{
    cd /etc/kubernetes/pki
    tar czvf kube-certs.tar.gz sa.pub sa.key ca.crt ca.key front-proxy-ca.crt front-proxy-ca.key etcd/ca.crt etcd/ca.key
    scp kube-certs.tar.gz ubuntu@192.168.219.101:/tmp
}
```

```bash
# k8s-master-1 node

$ kubeadm token create --print-join-command
kubeadm join 192.168.219.100:6443 --token nt5o96.s895kkb775ywed3b --discovery-token-ca-cert-hash sha256:058acc0a08802cd8d7aefeb0699ba8d8d66aeb79269278e31653e1af8998ef3e
```

<BR>

### B) Run join command you confirmed on a new Node with [--control-plane] option.

```bash
# k8s-master-2 node

# copy certificates transferred from existing Control Plane
$ mkdir /etc/kubernetes/pki
$ tar zxvf /tmp/kube-certs.tar.gz -C /etc/kubernetes/pki
$ kubeadm join 192.168.219.100:6443 --token nt5o96.s895kkb775ywed3b \
        --discovery-token-ca-cert-hash sha256:058acc0a08802cd8d7aefeb0699ba8d8d66aeb79269278e31653e1af8998ef3e \
        --cri-socket unix:///var/run/cri-dockerd.sock \
        --control-plane
```

<BR>

### C) Verify settings on Manager Node. That's OK if the status of new Node turns to [STATUS = Ready].

```bash
$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
k8s-master-1   Ready    control-plane   57m   v1.32.4
k8s-master-2   Ready    control-plane   10s   v1.32.4
k8s-worker01   Ready    <none>          55m   v1.32.4
k8s-worker02   Ready    <none>          55m   v1.32.4
```