# 0. 선행 작업

- [1. Installing Kubernetes with Docker Engine Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)

<br>

# 01. Add Worker Nodes

<BR>

### A) Confirm join command on Control Plane Node.

```bash
kubeadm token create --print-join-command
```

<BR>

### B) Run join command on a new Node.

```bash
kubeadm join 192.168.219.100:6443 --token nt5o96.s895kkb775ywed3b \
        --discovery-token-ca-cert-hash sha256:058acc0a08802cd8d7aefeb0699ba8d8d66aeb79269278e31653e1af8998ef3e \
        --cri-socket unix:///var/run/cri-dockerd.sock 
```

<BR>

### C) Verify settings on Manager Node. That's OK if the status of new Node turns to [STATUS = Ready].

```bash
$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   57m   v1.32.5
k8s-worker01   Ready    <none>          55m   v1.32.5
k8s-worker02   Ready    <none>          55m   v1.32.5
k8s-worker03   Ready    <none>          10s   v1.32.5
```