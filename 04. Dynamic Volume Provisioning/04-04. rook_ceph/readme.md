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
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1
├─sda2 ext4   1.0         3c586df5-01e3-4d9b-a728-24fc64ec26e6  810.2M    10% /boot
└─sda3 ext4   1.0         9b12b673-598d-48a1-b93a-b2acd64c2afc   13.1G    24% /
sdb ## <- There should be a storage device like this "sdb" with no configurations set.
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
# Create the custom resource definitions required for Rook operator and 
# deploy Rook operator resources to the Kubernetes cluster.
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

```bash
# Verify that the rook-ceph-operator pod is in a running state.
kubectl -n rook-ceph get pod
NAME                                 READY   STATUS    RESTARTS   AGE
rook-ceph-operator-6666d5d87-wqq69   1/1     Running   0          55s
```

```bash
# Create the Ceph cluster.
kubectl create -f cluster.yaml
```

```bash
# Verify that Ceph-related pods are in a running state.
kubectl -n rook-ceph get pod
NAME                                                     READY   STATUS      RESTARTS        AGE
csi-cephfsplugin-6m2sd                                   3/3     Running     1 (11m ago)     12m
csi-cephfsplugin-provisioner-64ff4dcc86-jqw8n            6/6     Running     1 (11m ago)     12m
csi-cephfsplugin-provisioner-64ff4dcc86-kxgpt            6/6     Running     1 (11m ago)     12m
csi-cephfsplugin-tnsvn                                   3/3     Running     1 (11m ago)     12m
csi-cephfsplugin-zvwbh                                   3/3     Running     1 (10m ago)     12m
csi-rbdplugin-d28mj                                      3/3     Running     1 (11m ago)     12m
csi-rbdplugin-hbr27                                      3/3     Running     1 (11m ago)     12m
csi-rbdplugin-jjsx9                                      3/3     Running     1 (11m ago)     12m
csi-rbdplugin-provisioner-7cbd54db94-bkcdz               6/6     Running     2 (9m13s ago)   12m
csi-rbdplugin-provisioner-7cbd54db94-fpq44               6/6     Running     1 (11m ago)     12m
rook-ceph-crashcollector-k8s-worker01-6dbccdc6fb-7z944   1/1     Running     0               5m25s
rook-ceph-crashcollector-k8s-worker02-768df65545-lj7ml   1/1     Running     0               4m48s
rook-ceph-crashcollector-k8s-worker03-8f88757db-mjjzf    1/1     Running     0               4m43s
rook-ceph-exporter-k8s-worker01-9df769445-scpw8          1/1     Running     0               5m25s
rook-ceph-exporter-k8s-worker02-79d776d444-qqlfx         1/1     Running     0               4m45s
rook-ceph-exporter-k8s-worker03-c5484b879-rpssk          1/1     Running     0               4m39s
rook-ceph-mgr-a-c84f87b8-tk5ls                           3/3     Running     0               5m27s
rook-ceph-mgr-b-fcf4b8b7f-l896w                          3/3     Running     0               5m27s
rook-ceph-mon-a-677f7f4d88-8pqjs                         2/2     Running     0               7m26s
rook-ceph-mon-b-68b6dd85c5-l6wm8                         2/2     Running     0               6m54s
rook-ceph-mon-c-6587c579f7-xh72c                         2/2     Running     0               5m39s
rook-ceph-operator-6666d5d87-wrxhp                       1/1     Running     0               12m
rook-ceph-osd-0-67b7f88b55-rsbs4                         2/2     Running     0               4m48s
rook-ceph-osd-1-7f45775d4d-fjflt                         2/2     Running     0               4m43s
rook-ceph-osd-2-7d9c8fcd6f-v9pc5                         2/2     Running     0               4m44s
rook-ceph-osd-prepare-k8s-worker01-zmpkm                 0/1     Completed   0               5m5s
rook-ceph-osd-prepare-k8s-worker02-twpnh                 0/1     Completed   0               5m4s
rook-ceph-osd-prepare-k8s-worker03-56cz5                 0/1     Completed   0               5m5s
```

```bash
# Install the Rook toolbox.
kubectl create -f toolbox.yaml
```

```bash
# Execute the `ceph status` command inside the Rook toolbox pod to check the Ceph cluster status.
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

bash-5.1$ ceph status
  cluster:
    id:     1adfb483-ae69-40a8-a419-3cd38cfa2541
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 6m)
    mgr: a(active, since 5m), standbys: b
    osd: 3 osds: 3 up (since 5m), 3 in (since 5m)

  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   80 MiB used, 60 GiB / 60 GiB avail
    pgs:     1 active+clean

bash-5.1$ ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME              STATUS  REWEIGHT  PRI-AFF
-1         0.05846  root default
-5         0.01949      host k8s-worker01
 2    hdd  0.01949          osd.2              up   1.00000  1.00000
-3         0.01949      host k8s-worker02
 0    hdd  0.01949          osd.0              up   1.00000  1.00000
-7         0.01949      host k8s-worker03
 1    hdd  0.01949          osd.1              up   1.00000  1.00000

bash-5.1$ ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL    USED  RAW USED  %RAW USED
hdd    60 GiB  60 GiB  80 MiB    80 MiB       0.13
TOTAL  60 GiB  60 GiB  80 MiB    80 MiB       0.13

--- POOLS ---
POOL  ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr   1    1  449 KiB        2  1.3 MiB      0     19 GiB

bash-5.1$ ceph osd df
ID  CLASS  WEIGHT   REWEIGHT  SIZE    RAW USE  DATA     OMAP     META    AVAIL   %USE  VAR   PGS  STATUS
 2    hdd  0.01949   1.00000  20 GiB   27 MiB  620 KiB    1 KiB  26 MiB  20 GiB  0.13  1.00    1      up
 0    hdd  0.01949   1.00000  20 GiB   27 MiB  620 KiB    1 KiB  26 MiB  20 GiB  0.13  1.00    1      up
 1    hdd  0.01949   1.00000  20 GiB   27 MiB  620 KiB    1 KiB  26 MiB  20 GiB  0.13  1.00    1      up
                       TOTAL  60 GiB   80 MiB  1.8 MiB  4.7 KiB  79 MiB  60 GiB  0.13
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

<BR>

### C) Final verification

```bash
# There is an examples directory in the source code cloned with `git clone`.
# Move to the examples directory and run the example app.
# Naturally, this is an example of receiving a PV from the Ceph cluster above.
cd rook/deploy/examples 

# Create the StorageClass that the example app will use.
kubectl apply -f csi/rbd/storageclass.yaml

kubectl create -f mysql.yaml
```

```bash
$ kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
mysql-pv-claim   Bound    pvc-f412b445-8d18-4937-869b-0b0d3daceec5   20Gi       RWO            rook-ceph-block   <unset>                 7s


$ kubectl exec -it wordpress-mysql-67d99b6d47-l2z2f -- df /var/lib/mysql
Filesystem     1K-blocks   Used Available Use% Mounted on
/dev/rbd0       20466256 118384  20331488   1% /var/lib/mysql
```

<br>

# 2. Ceph Dashboard

- [Prometheus Install](https://rook.io/docs/rook/latest-release/Storage-Configuration/Monitoring/ceph-monitoring/)
- [Dashboard Install](https://rook.io/docs/rook/latest-release/Storage-Configuration/Monitoring/ceph-dashboard/)

