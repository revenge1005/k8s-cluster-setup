# 0. 선행 작업

- [**Manager(Proxy,kubectl) Node**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-1.%20Manager(Proxy%2Ckubectl)%20Node)

<br>

# 01. Add Worker Nodes

![multi-node](https://github.com/revenge1005/k8s-cluster-setup/blob/main/multi-node-configuration.png)

<BR>

### A) Confirm join command on Control Plane Node.

```bash
$ kubeadm token create --print-join-command
kubeadm join 192.168.219.25:6443 --token 2jpwgc.29aq5nysp5pksicl --discovery-token-ca-cert-hash sha256:b94c81c60dcb7efa767a4e0650eb563062562afa6e394d303130fecd67f52612
```

<BR>

### B) Run join command on a new Node.

```bash
kubeadm join 192.168.219.25:6443 --token 2jpwgc.29aq5nysp5pksicl \
        --discovery-token-ca-cert-hash sha256:b94c81c60dcb7efa767a4e0650eb563062562afa6e394d303130fecd67f52612
```

<BR>

### C) Verify settings on Manager Node. That's OK if the status of new Node turns to [STATUS = Ready].

```bash
$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   57m   v1.32.4
k8s-worker01   Ready    <none>          55m   v1.32.4
k8s-worker02   Ready    <none>          55m   v1.32.4
k8s-worker03   Ready    <none>          10s   v1.32.4
```