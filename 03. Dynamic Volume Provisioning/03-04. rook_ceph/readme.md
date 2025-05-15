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

### A) 모든 worker node에 Raw device를 추가 장착 확인 (k8s-worker01, k8s-worker02, k8s-worker03)

```bash
$ lsblk -f
```

<BR>

### B) Rook을 활용해서 kubernetes에 ceph cluster 구축

- [Rook 공식 Github](https://github.com/rook/rook?tab=readme-ov-file)
- [Rook Quickstart Documentation](https://rook.github.io/docs/rook/latest-release/Getting-Started/quickstart/)

```bash
# GitHub에서 Rook operator를 내려 받는다.
git clone --single-branch --branch v1.17.2 https://github.com/rook/rook.git
 
cd rook/deploy/examples 
```

```bash
# Rook operator 운영에 필요한 custom resource definition을 생성하고 Rook operator 리소스를 kubernetes cluster에 배포한다.
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

```bash
# rook-ceph-operator pod가 running 상태인지 확인
kubectl -n rook-ceph get pod
```

```bash
# Ceph cluster를 생성
kubectl create -f cluster.yaml
```

```bash
# Ceph 관련 pod가 running 상태인지 확인
kubectl -n rook-ceph get pod
```

```bash
# Rook toolbox를 설치
kubectl create -f deploy/examples/toolbox.yaml
```

```bash
# Rook toolbox pod 내부에서 `ceph status` 명령을 수행하여 ceph cluster 상태를 조회
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```