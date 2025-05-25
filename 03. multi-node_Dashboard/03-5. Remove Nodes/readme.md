1. Remove Nodes

![multi-node](https://github.com/revenge1005/k8s-cluster-setup/blob/main/multi-node-configuration.png)

### A) Work on Manager node.

```bash
$ kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
k8s-master     Ready    control-plane   3m13s   v1.32.5
k8s-worker01   Ready    <none>          2m34s   v1.32.5
k8s-worker02   Ready    <none>          2m33s   v1.32.5
k8s-worker03   Ready    <none>          2m35s   v1.32.5
```

```bash
# prepare to remove a target node
# --ignore-daemonsets ⇒ ignore pods in DaemonSet
# --delete-emptydir-data ⇒ ignore pods that has emptyDir volumes
# --force ⇒ also remove pods that was created as a pod, not as deployment or others
kubectl drain k8s-worker03 --ignore-daemonsets --delete-emptydir-data --force
```

```bash
$ kubectl get nodes
NAME           STATUS                     ROLES           AGE     VERSION
k8s-master     Ready                      control-plane   5m27s   v1.32.5
k8s-worker01   Ready                      <none>          4m47s   v1.32.5
k8s-worker02   Ready                      <none>          4m46s   v1.32.5
k8s-worker03   Ready,SchedulingDisabled   <none>          4m48s   v1.32.5
```

```bash
# run delete method
kubectl delete node k8s-worker02
```

```bash
$ kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
k8s-master     Ready    control-plane   7m19s   v1.32.5
k8s-worker01   Ready    <none>          6m40s   v1.32.5
k8s-worker02   Ready    <none>          6m39s   v1.32.5
```

### B) On the removed Node, Reset kubeadm settings.

```bash
root@k8s-worker03:~# kubeadm reset
W0525 15:33:20.346885   15546 preflight.go:56] [reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
W0525 15:33:26.885906   15546 removeetcdmember.go:106] [reset] No kubeadm config, using etcd pod spec to get data directory
[reset] Deleted contents of the etcd data directory: /var/lib/etcd
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of directories: [/etc/kubernetes/manifests /var/lib/kubelet /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/super-admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
```