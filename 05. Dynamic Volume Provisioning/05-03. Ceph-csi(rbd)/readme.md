# 0. 선행 작업

- [1. Installing Kubernetes with Docker Engine Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)

or

- [2. Installing Kubernetes with containerd Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)

<br>

# 1. Kubernetes : Dynamic Volume Provisioning (Ceph-csi(cephfs))

- **Ceph Cluster**

```bash
            +----------------------------+----------------------------+
            |                            |                            |
            |192.168.219.51              |192.168.219.52              |192.168.219.53
+-----------+-----------+    +-----------+-----------+    +-----------+-----------+
|        [node01]       |    |        [node02]       |    |        [node03]       |
|     Object Storage    +----+     Object Storage    +----+     Object Storage    |
|     Monitor Daemon    |    |                       |    |                       |
|     Manager Daemon    |    |                       |    |                       |
+-----------------------+    +-----------------------+    +-----------------------+
```

- **ceph node setting:**

![ceph_cpu_memory](https://github.com/revenge1005/k8s-cluster-setup/blob/main/05.%20Dynamic%20Volume%20Provisioning/05-03.%20Ceph-csi(rbd)/ceph_cpu_memory.PNG)

<BR>

### A) Ceph : Configure Cluster #1 - node01

```bash
# Generate SSH key-pair on [Monitor Daemon] Node (call it Admin Node on here) and set it to each Node.
# Install Ceph to each Node from Admin Node.
{
	ssh-keygen -q -N ""
cat <<EOF >> /etc/hosts
192.168.219.51 node01 node01.test.srv
192.168.219.52 node02 node02.test.srv
192.168.219.53 node03 node03.test.srv
192.168.219.10 dlp
EOF
for NODE in node01 node02 node03
do
	ssh-copy-id $NODE
done
for NODE in node01 node02 node03
do
	ssh $NODE "apt update; apt -y install ceph"
done
}
```

```bash
# Configure [Monitor Daemon], [Manager Daemon] on Admin Node.
{
	export CEPH_UUID=$(uuidgen)
cat <<EOF > /etc/ceph/ceph.conf
[global]
cluster network = 192.168.219.0/24
public network = 192.168.219.0/24
fsid = $CEPH_UUID
mon host = 192.168.219.51
mon initial members = node01
osd pool default crush rule = -1

[mon.node01]
host = node01
mon addr = 192.168.219.51
mon allow pool delete = true
EOF
}
```

```bash
# Generate and configure keyrings for Ceph authentication
{
  # generate secret key for Cluster monitoring
  ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
  # generate secret key for Cluster admin
  ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
  # generate key for bootstrap
  ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'
  # import generated key
  ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
  ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
}
```

```bash
# generate monitor map
{
  FSID=$(grep "^fsid" /etc/ceph/ceph.conf | awk {'print $NF'})
  NODENAME=$(grep "^mon initial" /etc/ceph/ceph.conf | awk {'print $NF'})
  NODEIP=$(grep "^mon host" /etc/ceph/ceph.conf | awk {'print $NF'})
  monmaptool --create --add $NODENAME $NODEIP --fsid $FSID /etc/ceph/monmap
}
```

```bash
# Initialize and configure the Monitor and Manager Daemons
{
  # create a directory for Monitor Daemon
  # directory name ⇒ (Cluster Name)-(Node Name)
  mkdir /var/lib/ceph/mon/ceph-node01
  ceph-mon --cluster ceph --mkfs -i $NODENAME --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring
  chown ceph:ceph /etc/ceph/ceph.*
  chown -R ceph:ceph /var/lib/ceph/mon/ceph-node01 /var/lib/ceph/bootstrap-osd
  systemctl enable --now ceph-mon@$NODENAME
  # enable Messenger v2 Protocol
  ceph mon enable-msgr2
  ceph config set mon auth_allow_insecure_global_id_reclaim false
  # enable Placement Groups auto scale module
  ceph mgr module enable pg_autoscaler
  # create a directory for Manager Daemon
  # directory name ⇒ (Cluster Name)-(Node Name)
  mkdir /var/lib/ceph/mgr/ceph-node01
  # create auth key
  ceph auth get-or-create mgr.$NODENAME mon 'allow profile mgr' osd 'allow *' mds 'allow *'
}
```

```bash
# Configure the Manager Daemon keyring and service
{
  ceph auth get-or-create mgr.node01 | tee /etc/ceph/ceph.mgr.admin.keyring
  cp /etc/ceph/ceph.mgr.admin.keyring /var/lib/ceph/mgr/ceph-node01/keyring
  chown ceph:ceph /etc/ceph/ceph.mgr.admin.keyring
  chown -R ceph:ceph /var/lib/ceph/mgr/ceph-node01
  systemctl enable --now ceph-mgr@$NODENAME
}
```

```bash
$ ceph -s
  cluster:
    id:     c7e23d20-36e7-462d-b2b2-dbadc96b52e7
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum node01 (age 14s)
    mgr: node01(active, since 3s)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

<BR>

### B) Ceph : Configure Cluster #2 - node01

```bash
# Configure OSD (Object Storage Device) to each Node from Admin Node.
for NODE in node01 node02 node03
do
    if [ ! ${NODE} = "node01" ]
    then
        scp /etc/ceph/ceph.conf ${NODE}:/etc/ceph/ceph.conf
        scp /etc/ceph/ceph.client.admin.keyring ${NODE}:/etc/ceph
        scp /var/lib/ceph/bootstrap-osd/ceph.keyring ${NODE}:/var/lib/ceph/bootstrap-osd
    fi
    ssh $NODE \
    "chown ceph:ceph /etc/ceph/ceph.* /var/lib/ceph/bootstrap-osd/*; \
    parted --script /dev/sdb 'mklabel gpt'; \
    parted --script /dev/sdb "mkpart primary 0% 100%"; \
    ceph-volume lvm create --data /dev/sdb1"
done
```

```bash
# Display the overall status of the Ceph cluster
$ ceph -s
  cluster:
    id:     c7e23d20-36e7-462d-b2b2-dbadc96b52e7
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum node01 (age 2m)
    mgr: node01(active, since 2m)
    mds: 1/1 daemons up
    osd: 3 osds: 3 up (since 70s), 3 in (since 78s)

  data:
    volumes: 1/1 healthy
    pools:   3 pools, 145 pgs
    objects: 24 objects, 461 KiB
    usage:   888 MiB used, 59 GiB / 60 GiB avail
    pgs:     97.931% pgs unknown
             142 unknown
             3   active+clean

  progress:
    Global Recovery Event (0s)
      [............................]


# Display the hierarchical structure of OSDs in the Ceph cluster
$ ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME        STATUS  REWEIGHT  PRI-AFF
-1         0.05846  root default
-3         0.01949      host node01
 0    hdd  0.01949          osd.0        up   1.00000  1.00000
-5         0.01949      host node02
 1    hdd  0.01949          osd.1        up   1.00000  1.00000
-7         0.01949      host node03
 2    hdd  0.01949          osd.2        up   1.00000  1.00000


# Display storage usage summary for the Ceph cluster
$ ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL     USED  RAW USED  %RAW USED
hdd    60 GiB  59 GiB  897 MiB   897 MiB       1.46
TOTAL  60 GiB  59 GiB  897 MiB   897 MiB       1.46

--- POOLS ---
POOL                    ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr                     1    1  449 KiB        2  1.3 MiB      0     19 GiB
cephfs.kubernetes.meta   2   16   12 KiB       22  120 KiB      0     19 GiB
cephfs.kubernetes.data   3  128      0 B        0      0 B      0     19 GiB


# Display detailed storage usage for each OSD
$ ceph osd df
ID  CLASS  WEIGHT   REWEIGHT  SIZE    RAW USE  DATA     OMAP  META     AVAIL   %USE  VAR   PGS  STATUS
 0    hdd  0.01949   1.00000  20 GiB  299 MiB  940 KiB   0 B  298 MiB  20 GiB  1.46  1.00  145      up
 1    hdd  0.01949   1.00000  20 GiB  299 MiB  940 KiB   0 B  298 MiB  20 GiB  1.46  1.00  145      up
 2    hdd  0.01949   1.00000  20 GiB  299 MiB  936 KiB   0 B  298 MiB  20 GiB  1.46  1.00  145      up
                       TOTAL  60 GiB  897 MiB  2.8 MiB   0 B  894 MiB  59 GiB  1.46
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

```bash
{
  # create Kubernetes pool [rbd]
  ceph osd pool create kubernetes
  # initialize the pool
  rbd pool init kubernetes
  # Create the user for Kubernetes
  ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
  ceph osd pool autoscale-status
  ceph mon dump
}
```

<BR>

# 2. k8s Setting

### A) ceph-csi (rbd) Configure (k8s-master)

```bash
# ConfigMap with CSI configuration.
# The clusterID is obtained from the 'ceph -s' command on node01 of the Ceph cluster
cat <<EOF > csi-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "e6816ea6-d91c-40c4-b5e7-e6d27729e400",
        "monitors": [
          "192.168.219.51:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF

kubectl create -f csi-config-map.yaml
```

```bash
# ConfigMap with KMS configuration.
cat <<EOF > csi-kms-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
EOF

kubectl create -f csi-kms-config-map.yaml
```

```bash
# Another config map required for Ceph CSI (I don't remember why)
cat <<EOF > ceph-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
EOF

kubectl create -f ceph-config-map.yaml
```

```bash
# Secret with the user and key created before.
# The userKey is obtained from the Ceph cluster's management node using the "ceph auth list" command
cat <<EOF > csi-rbd-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: kubernetes
  userKey: AQB+ayRoJMB0BhAAAy6hKs2qPxV2c/W7Fk+RYQ==
EOF

kubectl create -f csi-rbd-secret.yaml
```

```bash
# Create the Roles
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
```

```bash
# Deploy the provisioner and the node plugin
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
kubectl apply -f csi-rbdplugin-provisioner.yaml
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin.yaml
kubectl apply -f csi-rbdplugin.yaml
```

```bash
$ kubectl get all
NAME                                             READY   STATUS    RESTARTS   AGE
pod/csi-rbdplugin-57ztr                          3/3     Running   0          2m22s
pod/csi-rbdplugin-p9t5j                          3/3     Running   0          2m22s
pod/csi-rbdplugin-provisioner-74c5fc75db-jchsh   0/7     Pending   0          2m22s
pod/csi-rbdplugin-provisioner-74c5fc75db-rn2gz   7/7     Running   0          2m22s
pod/csi-rbdplugin-provisioner-74c5fc75db-s6994   7/7     Running   0          2m22s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/csi-metrics-rbdplugin       ClusterIP   10.106.8.173    <none>        8080/TCP   2m22s
service/csi-rbdplugin-provisioner   ClusterIP   10.97.230.101   <none>        8080/TCP   2m22s
service/kubernetes                  ClusterIP   10.96.0.1       <none>        443/TCP    12m

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/csi-rbdplugin   2         2         2       2            2           <none>          2m22s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-rbdplugin-provisioner   2/3     3            2           2m22s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-rbdplugin-provisioner-74c5fc75db   3         3         2       2m22s
```

### C) StorageClass Configuration and Deployment 

```bash
# Create a StorageClass configuration for CephFS using the Ceph CSI driver
cat <<EOF > csi-rbd-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: e6816ea6-d91c-40c4-b5e7-e6d27729e400
   pool: kubernetes
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF

kubectl apply -f csi-rbd-sc.yaml
```

### D) Final verification

```bash
# Create a PVC to be used as raw block device (volumeMode: Block)
cat <<EOF > raw-block-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF

kubectl apply -f raw-block-pvc.yaml
```

```bash
# Create a new PVC now to be used as FS (volumeMode: Filesystem)
cat <<EOF > pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF

kubectl apply -f pvc.yaml
```

```bash
$ kubectl get sc,pvc,pv
NAME                                     PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/csi-rbd-sc   rbd.csi.ceph.com   Delete          Immediate           true                   7m13s

NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/raw-block-pvc   Bound    pvc-a7a3728e-fd22-433b-8245-49be121cb24b   1Gi        RWO            csi-rbd-sc     <unset>                 7m3s
persistentvolumeclaim/rbd-pvc         Bound    pvc-9d2deabf-5301-4af1-afb5-4ac3215b4f6b   1Gi        RWO            csi-rbd-sc     <unset>                 6s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-9d2deabf-5301-4af1-afb5-4ac3215b4f6b   1Gi        RWO            Delete           Bound    default/rbd-pvc         csi-rbd-sc     <unset>                          6s
persistentvolume/pvc-a7a3728e-fd22-433b-8245-49be121cb24b   1Gi        RWO            Delete           Bound    default/raw-block-pvc   csi-rbd-sc     <unset>                          7m3s
```

```bash
# Create a pod with the PVC as a raw block device
cat <<EOF > raw-block-pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-raw-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: ["tail -f /dev/null"]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: raw-block-pvc
EOF

kubectl apply -f raw-block-pod.yaml
```

```bash
# Create a pod and mount the PVC as a FS
cat <<EOF > pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-rbd-demo-pod
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false
EOF

kubectl apply -f pod.yaml
```

```bash
$ kubectl get pod,pvc,pv
NAME                                             READY   STATUS    RESTARTS   AGE
pod/csi-rbd-demo-pod                             1/1     Running   0          30s
pod/csi-rbdplugin-bl26x                          3/3     Running   0          8m54s
pod/csi-rbdplugin-provisioner-74c5fc75db-pzl58   7/7     Running   0          8m55s
pod/csi-rbdplugin-provisioner-74c5fc75db-q6pjn   0/7     Pending   0          8m55s
pod/csi-rbdplugin-provisioner-74c5fc75db-x5qdl   7/7     Running   0          8m55s
pod/csi-rbdplugin-ptjjv                          3/3     Running   0          8m54s
pod/pod-with-raw-block-volume                    1/1     Running   0          115s

NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/raw-block-pvc   Bound    pvc-a7a3728e-fd22-433b-8245-49be121cb24b   1Gi        RWO            csi-rbd-sc     <unset>                 8m4s
persistentvolumeclaim/rbd-pvc         Bound    pvc-9d2deabf-5301-4af1-afb5-4ac3215b4f6b   1Gi        RWO            csi-rbd-sc     <unset>                 67s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-9d2deabf-5301-4af1-afb5-4ac3215b4f6b   1Gi        RWO            Delete           Bound    default/rbd-pvc         csi-rbd-sc     <unset>                          67s
persistentvolume/pvc-a7a3728e-fd22-433b-8245-49be121cb24b   1Gi        RWO            Delete           Bound    default/raw-block-pvc   csi-rbd-sc     <unset>                          8m4s

$ kubectl exec pod-with-raw-block-volume -- lsblk | grep rbd0
rbd0   251:0    0    1G  0 disk

$ kubectl exec csi-rbd-demo-pod -- df /var/lib/www/html
Filesystem     1K-blocks  Used Available Use% Mounted on
/dev/rbd0         996780    24    980372   1% /var/lib/www/html
```