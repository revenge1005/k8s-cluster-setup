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

### A) Ceph : Configure Cluster #1

```bash
# Generate SSH key-pair on [Monitor Daemon] Node (call it Admin Node on here) and set it to each Node.
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
$ {
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

{
    # create a directory for Monitor Daemon
    # directory name ⇒ (Cluster Name)-(Node Name)
	mkdir /var/lib/ceph/mon/ceph-node01
	ceph-mon --cluster ceph --mkfs -i $NODENAME --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring
	chown ceph. /etc/ceph/ceph.*
	chown -R ceph. /var/lib/ceph/mon/ceph-node01 /var/lib/ceph/bootstrap-osd
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

{
	ceph auth get-or-create mgr.node01 | tee /etc/ceph/ceph.mgr.admin.keyring
	cp /etc/ceph/ceph.mgr.admin.keyring /var/lib/ceph/mgr/ceph-node01/keyring
	chown ceph. /etc/ceph/ceph.mgr.admin.keyring
	chown -R ceph. /var/lib/ceph/mgr/ceph-node01
	systemctl enable --now ceph-mgr@$NODENAME
}
```

```bash
$ ceph -s
  cluster:
    id:     3666a474-14e0-4c5f-ad1e-daf2e30aed8f
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 1 daemons, quorum node01 (age 117s)
    mgr: node01(active, since 25s)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

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
    "chown ceph. /etc/ceph/ceph.* /var/lib/ceph/bootstrap-osd/*; \
    parted --script /dev/sdb 'mklabel gpt'; \
    parted --script /dev/sdb "mkpart primary 0% 100%"; \
    ceph-volume lvm create --data /dev/sdb1"
done 
```

```bash

```

```bash
# 
{
	mkdir -p /var/lib/ceph/mds/ceph-node01
	ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-node01/keyring --gen-key -n mds.node01
creating /var/lib/ceph/mds/ceph-node01/keyring
	chown -R ceph:ceph /var/lib/ceph/mds/ceph-node01
	ceph auth add mds.node01 osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-node01/keyring
	systemctl enable --now ceph-mds@node01
}

# 
{
	ceph fs volume create kubernetes
	ceph fs ls
	ceph auth get-or-create client.cephfs mon 'allow r' osd 'allow rwx pool=kubernetes'
	ceph mds stat
	ceph fs subvolumegroup create kubernetes csi
	ceph fs subvolumegroup ls kubernetes
}	
```