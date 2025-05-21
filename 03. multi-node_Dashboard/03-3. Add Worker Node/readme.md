# 0. 선행 작업

- [Manager(Proxy,kubectl) Node](https://github.com/revenge1005/k8s-cluster-setup/tree/main/03.%20multi-node_Dashboard/03-1.%20Manager(Proxy%2Ckubectl)%20Node)

<br>

# 01. Add Worker Nodes

![multi-node](https://github.com/revenge1005/k8s-cluster-setup/blob/main/multi-node-configuration.png)

<BR>

### A) Confirm join command on Control Plane Node.

```bash
$ kubeadm token create --print-join-command
kubeadm join 10.0.0.25:6443 --token pqsfr3.5zt0ffe08xi1iz4x --discovery-token-ca-cert-hash sha256:9cb21e3807780e1acf9ad9b6369ce54b1141ecf007d099675b23b6c3368494c9
```

<BR>

### B) Run join command on a new Node.

```bash
kubeadm join 192.168.219.100:6443 --token pqsfr3.5zt0ffe08xi1iz4x \
        --discovery-token-ca-cert-hash sha256:9cb21e3807780e1acf9ad9b6369ce54b1141ecf007d099675b23b6c3368494c9 
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