# 0. 선행 작업

- [**Installing Kubernetes with containerd Runtime**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)

- [**Kubernetes : Dynamic Volume Provisioning (NFS)**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/05.%20Dynamic%20Volume%20Provisioning/05-01.%20NFS)

<BR>

# 1. Monitoring(Prometheus,Grafana)

- **k8s cluster + NFS Server**

```
+----------------------+   +----------------------+
|  [ Dynamic Volume    |   |                      |
|     Provisioning ]   |   |     [ k8s-master ]   |
|     NFS Server       |   |     Control Plane    |
+-----------+----------+   +-----------+----------+
        eth0|192.168.219.30        eth0|192.168.219.100
            |                          |
------------+--------------------------+-----------
            |                          |
        eth0|192.168.219.110       eth0|192.168.219.120
+-----------+----------+   +-----------+----------+
|   [ k8s-worker01 ]   |   |   [ k8s-worker02 ]   |
|     Worker Node#1    |   |     Worker Node#2    |
+----------------------+   +----------------------+
```

### A) Install Prometheus chart with Helm.

```bash
# Install Helm.
{
    curl -O https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    bash ./get-helm-3
    helm version
}
```

```bash
# output config and change some settings
helm repo add bitnami https://charts.bitnami.com/bitnami

helm inspect values bitnami/kube-prometheus > prometheus.yaml
```

```bash
$ vi prometheus.yaml

.....
.....
    # line 21 : specify [storageClass] to use
    storageClass: "nfs-client"
.....
.....
.....
    # line 1249 : specify [storageClass] to use
    storageClass: "nfs-client"
.....
.....
.....
    # line 2327 : specify [storageClass] to use
    storageClass: "nfs-client"
```

```bash
# create a namespace for Prometheus
kubectl create namespace monitoring

helm install prometheus --namespace monitoring -f prometheus.yaml bitnami/kube-prometheus
```

```bash
$ kubectl get pods -n monitoring
NAME                                                            READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0          2/2     Running   0          104s
prometheus-kube-prometheus-blackbox-exporter-559b9479ff-vpvmh   1/1     Running   0          2m22s
prometheus-kube-prometheus-operator-5f9dccf6bc-jlnjf            1/1     Running   0          2m22s
prometheus-kube-state-metrics-74cf974698-zlkrc                  1/1     Running   0          2m22s
prometheus-node-exporter-56rmf                                  1/1     Running   0          2m22s
prometheus-node-exporter-kdpcr                                  1/1     Running   0          2m22s
prometheus-prometheus-kube-prometheus-prometheus-0              2/2     Running   0          104s
```

```bash
# if access from outside of cluster, set port-forwarding
kubectl port-forward -n monitoring service/prometheus-kube-prometheus-prometheus --address 0.0.0.0 9090:9090 &
```

### B) If you deploy Grafana, too, It's possible like follows.

```bash
# output config and change some settings
helm inspect values bitnami/grafana > grafana.yaml
```

```bash
vi grafana.yaml

# line 612 : change to your [storageClass]
persistence:
  enabled: true
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  storageClass: "nfs-client"
```

```bash
helm install grafana --namespace monitoring -f grafana.yaml bitnami/grafana
```

```bash
$ kubectl get pods -n monitoring
NAME                                                            READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0          2/2     Running   0          4m31s
grafana-6655c7ff45-cpl7v                                        1/1     Running   0          97s  # <- grafana
prometheus-kube-prometheus-blackbox-exporter-559b9479ff-vpvmh   1/1     Running   0          5m9s
prometheus-kube-prometheus-operator-5f9dccf6bc-jlnjf            1/1     Running   0          5m9s
prometheus-kube-state-metrics-74cf974698-zlkrc                  1/1     Running   0          5m9s
prometheus-node-exporter-56rmf                                  1/1     Running   0          5m9s
prometheus-node-exporter-kdpcr                                  1/1     Running   0          5m9s
prometheus-prometheus-kube-prometheus-prometheus-0              2/2     Running   0          4m31s
```

```bash
# if access from outside of cluster, set port-forwarding
kubectl port-forward -n monitoring service/grafana --address 0.0.0.0 3000:3000 &
```

### C) prometheus - Access to the URL below on a client computer in your local network.

- https://(Control-plan Node IP address):9090/

or

- If you have configured multi nodes, https://(Manager(porxy,admin) Node IP address):9090/

![prometheus-1](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/prometheus-1.PNG)

![prometheus-2](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/prometheus-2.PNG)

### D) Grafana - Access to the URL below on a client computer in your local network.

- https://(Control-plan Node IP address):3000/

or

- If you have configured multi nodes, https://(Manager(porxy,admin) Node IP address):3000/

```bash
# admin password
$ kubectl get secret grafana-admin --namespace monitoring -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 -d
PskbWLsRrl
```

![Grafana-1](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-1.PNG)

![Grafana-2](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-2.PNG)

![Grafana-3](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-3.PNG)

![Grafana-4](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-4.PNG)

![Grafana-5](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-5.PNG)

![Grafana-6](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-6.PNG)

![Grafana-7](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-7.PNG)

![Grafana-8](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-8.PNG)

![Grafana-9](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/grafana-9.PNG)