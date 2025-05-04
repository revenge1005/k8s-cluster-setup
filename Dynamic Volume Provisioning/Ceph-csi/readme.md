## 00. 선행 작업

- [1. Installing Kubernetes with Docker Engine Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/Container%20runtime/01.%20Docker%20Engine)

or

- [2. Installing Kubernetes with containerd Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/Container%20runtime/02.%20containerd)

<br>

## 01. Kubernetes : Dynamic Volume Provisioning (Ceph-csi)

```bash
# Ceph Cluster
            +----------------------------+----------------------------+
            |                            |                            |
            |192.168.219.51              |192.168.219.52              |192.168.219.53
+-----------+-----------+    +-----------+-----------+    +-----------+-----------+
|        [node01]       |    |        [node01]       |    |        [node01]       |
|     Object Storage    +----+     Object Storage    +----+     Object Storage    |
|     Monitor Daemon    |    |                       |    |                       |
|     Manager Daemon    |    |                       |    |                       |
+-----------------------+    +-----------------------+    +-----------------------+
```

<BR>

### A) Ceph : Configure Cluster #1

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


# generate monitor map
{
  FSID=$(grep "^fsid" /etc/ceph/ceph.conf | awk {'print $NF'})
  NODENAME=$(grep "^mon initial" /etc/ceph/ceph.conf | awk {'print $NF'})
  NODEIP=$(grep "^mon host" /etc/ceph/ceph.conf | awk {'print $NF'})
  monmaptool --create --add $NODENAME $NODEIP --fsid $FSID /etc/ceph/monmap
}


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

### B) Ceph : Configure Cluster #2

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
# Block for configuring Ceph MDS (Metadata Server)
{
	# Create the directory /var/lib/ceph/mds/ceph-node01 (if it doesn't exist)
	mkdir -p /var/lib/ceph/mds/ceph-node01
	
	# Create a keyring file for MDS and generate a key for mds.node01
	ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-node01/keyring --gen-key -n mds.node01
	
	# Change ownership of the created directory and files to the ceph user and group
	chown -R ceph:ceph /var/lib/ceph/mds/ceph-node01
	
	# Add permissions for the mds.node01 entity:
	# - OSD: read/write/execute permissions
	# - MDS: full permissions
	# - MON: MDS profile permissions
	# Use the keyring file for authentication
	ceph auth add mds.node01 osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-node01/keyring
	
	# Enable and start the ceph-mds@node01 service immediately
	systemctl enable --now ceph-mds@node01
}
```

```bash
# Create 2 RADOS pools for Data and Metadata on the MDS Node.
# Refer to the official documentation to specify the PG (Placement Group) number (64 in the example below).
{
	# Create a RADOS pool named cephfs_data for data storage with 32 PGs
	ceph osd pool create cephfs_data 32

	# Create a RADOS pool named cephfs_metadata for metadata storage with 32 PGs
	ceph osd pool create cephfs_metadata 32

	# Enable the bulk flag on the cephfs_data pool to optimize for large data workloads
	ceph osd pool set cephfs_data bulk true

	# Create a new Ceph filesystem named cephfs, associating it with the metadata and data pools
	ceph fs new cephfs cephfs_metadata cephfs_data

	# List all Ceph filesystems in the cluster
	ceph fs ls

	# Check the status of the Metadata Server (MDS) to ensure it is running correctly
	ceph mds stat

	# Display detailed status of the cephfs filesystem, including pool, MDS, and client information
	ceph fs status cephfs
}
```

<BR>

### C) ceph-csi Configure (k8s-master)

```bash
# Install Helm, refer to here.
curl -O https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
bash ./get-helm-3
```

```bash
# Add the Ceph CSI Helm chart repository to the local Helm configuration
helm repo add ceph-csi https://ceph.github.io/csi-charts

# Search for available charts in the ceph-csi repository
helm search repo ceph-csi
```

```bash
# Create a configuration file for ceph-csi to connect to the Ceph cluster
# The clusterID is obtained from the 'ceph -s' command on node01 of the Ceph cluster
cat <<EOF > ceph-csi.vo
csiConfig:
- clusterID: "c7e23d20-36e7-462d-b2b2-dbadc96b52e7"
  monitors:
  - "192.168.219.51:6789"
EOF

# Create a dedicated Kubernetes namespace for ceph-csi-cephfs components
{
    kubectl create namespace "ceph-csi-cephfs"
    # Install the ceph-csi-cephfs Helm chart in the specified namespace
    # The chart integrates CephFS with Kubernetes via the CSI driver
    # The -f flag applies the configuration from ceph-csi.vo
    # Note: Installation may take some time depending on the test environment
    helm install --namespace "ceph-csi-cephfs" "ceph-csi-cephfs" ceph-csi/ceph-csi-cephfs -f ceph-csi.vo
    watch kubectl get all -n ceph-csi-cephfs
}
```

```bash
$ kubectl get all -n ceph-csi-cephfs                                                                                                

NAME                                              READY   STATUS    RESTARTS   AGE
pod/ceph-csi-cephfs-nodeplugin-cdqkj              3/3     Running   0          13m
pod/ceph-csi-cephfs-nodeplugin-xtfv2              3/3     Running   0          13m
pod/ceph-csi-cephfs-provisioner-bff7587bd-72zwk   5/5     Running   0          13m
pod/ceph-csi-cephfs-provisioner-bff7587bd-ckqmn   5/5     Running   0          13m
pod/ceph-csi-cephfs-provisioner-bff7587bd-m2kqc   0/5     Pending   0          13m

NAME                                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/ceph-csi-cephfs-nodeplugin-http-metrics    ClusterIP   10.103.142.135   <none>        8080/TCP   13m
service/ceph-csi-cephfs-provisioner-http-metrics   ClusterIP   10.105.138.116   <none>        8080/TCP   13m

NAME                                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/ceph-csi-cephfs-nodeplugin   2         2         2       2            2           <none>          13m

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ceph-csi-cephfs-provisioner   2/3     3            2           13m

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/ceph-csi-cephfs-provisioner-bff7587bd   3         3         2       13m
```

```bash
# The userKey is obtained from the Ceph cluster's management node using the "ceph auth list" command
cat <<EOF > csi-fs-secret.yaml 
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-fs-secret
  namespace: kube-system
stringData:
  userID: cephfs
  userKey: AQBeHA9oWD13KhAAzhysDMznCUfKhAEcjq5QTw==
  adminID: admin
  adminKey: AQD0Gw9ooEluMhAAntscL1ae3t62BYikI7+0pQ==
EOF

kubectl apply -f csi-fs-secret.yaml 
```

### D) StorageClass Configuration and Deployment 

```bash
# Create a StorageClass configuration for CephFS using the Ceph CSI driver
cat <<EOF > csi-fs-sc.yaml 
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-fs-sc
allowVolumeExpansion: true
provisioner: cephfs.csi.ceph.com
parameters:
   clusterID: c7e23d20-36e7-462d-b2b2-dbadc96b52e7
   fsName: kubernetes
   csi.storage.k8s.io/controller-expand-secret-name: csi-fs-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: kube-system
   csi.storage.k8s.io/provisioner-secret-name: csi-fs-secret
   csi.storage.k8s.io/provisioner-secret-namespace: kube-system
   csi.storage.k8s.io/node-stage-secret-name: csi-fs-secret
   csi.storage.k8s.io/node-stage-secret-namespace: kube-system
reclaimPolicy: Delete
mountOptions:
   - debug
EOF

kubectl apply -f csi-fs-sc.yaml 
```

```bash
# Create and Deploy pvc
cat <<EOF > fs-pvc.yaml 
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-cephfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-fs-sc
EOF

kubectl apply -f fs-pvc.yaml 
```

### E) 확인

```bash
$ kubectl get sc,pvc,pv

```