# 0. 선행 작업

- [1. Installing Kubernetes with Docker Engine Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)

or

- [2. Installing Kubernetes with containerd Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)

<br>

# 1. Kubernetes : Dynamic Volume Provisioning (NFS)


```
+----------------------+   +----------------------+
|  [ Dynamic Volume    |   |                      |
|     Provisioning ]   |   |     [ k8s-master ]   |
|     NFS Server       |   |     Control Plane    |
+-----------+----------+   +-----------+----------+
        eth0|192.168.219.30        eth0|192.168.219.10
            |                          |
------------+--------------------------+-----------
            |                          |
        eth0|192.168.219.11        eth0|192.168.219.12
+-----------+----------+   +-----------+----------+
|   [ k8s-worker01 ]   |   |   [ k8s-worker02 ]   |
|     Worker Node#1    |   |     Worker Node#2    |
+----------------------+   +----------------------+
```

### A) Configure NFS Server.

```bash
apt -y install nfs-kernel-server


cat <<EOF >> /etc/exports
/home/nfsshare 192.168.219.0/24(rw,no_root_squash)
EOF


mkdir /home/nfsshare
systemctl restart nfs-server
```

### B) 'Worker Nodes' need to be able to mount NFS share on NFS server. (k8s-worker01, k8s-worker02)

```bash
apt -y install nfs-common
```

### C) Install Helm, refer to here. (k8s-master)

```bash
curl -O https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

bash ./get-helm-3

helm version
```

### D) Install NFS Client Provisioner.

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# nfs.server = (NFS server's hostname or IP address)
# nfs.path = (NFS share Path)
helm install nfs-client -n kube-system --set nfs.server=192.168.219.30 --set nfs.path=/home/nfsshare nfs-subdir-external-provisioner/nfs-subdir-external-provisioner
```

```bash
$ kubectl get deployment -n kube-system
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
calico-kube-controllers                      1/1     1            1           4h54m
coredns                                      2/2     2            2           4h55m
metrics-server                               1/1     1            1           16m
nfs-client-nfs-subdir-external-provisioner   1/1     1            1           19s
```

<br>

# 2. This is an example to use dynamic volume provisioning by a Pod.

```bash
cat <<EOF >  my-pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-provisioner
spec:
  accessModes:
    - ReadWriteOnce
  # specify StorageClass name
  storageClassName: nfs-client
  resources:
    requests:
      # volume size
      storage: 5Gi
EOF
```

```bash
$ kubectl apply -f my-pvc.yml
persistentvolumeclaim/my-provisioner created
root@ctrl:~# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
my-provisioner   Bound    pvc-7b9f17ed-fc6f-4e7e-99dc-96324c1a4076   5Gi        RWO            nfs-client     <unset>                 5s

# PV is generated dynamically
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-7b9f17ed-fc6f-4e7e-99dc-96324c1a4076   5Gi        RWO            Delete           Bound    default/my-provisioner   nfs-client     <unset>                          35s
```

```bash
cat <<EOF > my-pod.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-pvc
      volumes:
        - name: nginx-pvc
          persistentVolumeClaim:
            # PVC name you created
            claimName: my-provisioner
EOF
```

```bash
$ kubectl apply -f my-pod.yml
deployment.apps/my-nginx created

$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
my-nginx-69d9cb4f47-zzjj7   1/1     Running   0          5s

$ kubectl exec my-nginx-69d9cb4f47-zzjj7 -- df /usr/share/nginx/html
Filesystem                                                                                    1K-blocks  Used Available Use% Mounted on
192.168.219.30:/home/nfsshare/default-my-provisioner-pvc-7b9f17ed-fc6f-4e7e-99dc-96324c1a4076 164028416     0 155623424   0% /usr/share/nginx/html
```