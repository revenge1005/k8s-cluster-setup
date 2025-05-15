# 0. 선행 작업

- [1. Installing Kubernetes with Docker Engine Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/Container%20runtime/01.%20Docker%20Engine)

or

- [2. Installing Kubernetes with containerd Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/Container%20runtime/02.%20containerd)

<br>

# 1. Kubernetes : Dynamic Volume Provisioning (Rook Ceph)

- **k8s Cluster**

```bash
+----------------------+   +----------------------+
|   [k8s-master]       |   |    [k8s-worker01]    |
|   Control Plane      |   |     Worker Node#1    |
+-----------+----------+   +-----------+----------+
        eth0|192.168.219.100     eth0|192.168.219.110
            |                          |
------------+--------------------------+-----------
            |                          |
        eth0|192.168.219.120     eth0|192.168.219.130
+-----------+----------+   +-----------+----------+
|    [k8s-worker02]    |   |    [k8s-worker03]    |
|     Worker Node#2    |   |     Worker Node#3    |
+-----------+----------+   +-----------+----------+
```

- **worker node(k8s-worker01, k8s-worker02, k8s-worker03) setting:**

![ceph_cpu_memory](https://github.com/revenge1005/k8s-cluster-setup/blob/main/03.%20Dynamic%20Volume%20Provisioning/03-04.%20rook_ceph/rook_ceph_cpu_memoy.PNG)

<BR>

### A) Verify that a Raw Device is Added to All Worker Nodes (k8s-worker01, k8s-worker02, k8s-worker03)

```bash
$ lsblk -f
```

<BR>

### B) Deploy a Ceph Cluster on Kubernetes Using Rook

- [Rook Github](https://github.com/rook/rook?tab=readme-ov-file)
- [Rook Quickstart Documentation](https://rook.github.io/docs/rook/latest-release/Getting-Started/quickstart/)

```bash
# Download the Rook operator from GitHub.
git clone --single-branch --branch v1.17.2 https://github.com/rook/rook.git
 
cd rook/deploy/examples 
```

```bash
# Create the custom resource definitions required for Rook operator and deploy Rook operator resources to the Kubernetes cluster.
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

```bash
# Verify that the rook-ceph-operator pod is in a running state.
kubectl -n rook-ceph get pod
```

```bash
# Create the Ceph cluster.
kubectl create -f cluster.yaml
```

```bash
# Verify that Ceph-related pods are in a running state.
kubectl -n rook-ceph get pod
```

```bash
# Install the Rook toolbox.
kubectl create -f deploy/examples/toolbox.yaml
```

```bash
# Execute the `ceph status` command inside the Rook toolbox pod to check the Ceph cluster status.
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```