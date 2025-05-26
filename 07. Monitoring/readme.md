# 0. 선행 작업

- [**Installing Kubernetes with containerd Runtime**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)

- [**Kubernetes : Dynamic Volume Provisioning (NFS)**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/05.%20Dynamic%20Volume%20Provisioning/05-01.%20NFS)

<BR>

# 1. Monitoring(Prometheus,Grafana,Loki)

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

![prometheus-1](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/prometheus-1.PNG)

![prometheus-2](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/prometheus-2.PNG)

### D) Grafana - Access to the URL below on a client computer in your local network.

- https://(Control-plan Node IP address):3000/

or

- If you have configured multi nodes, https://(Manager(porxy,admin) Node IP address):3000/

```bash
# admin password
$ kubectl get secret grafana-admin --namespace monitoring -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 -d
PskbWLsRrl
```

![Grafana-1](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-1.PNG)

![Grafana-2](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-2.PNG)

![Grafana-3](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-3.PNG)

![Grafana-4](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-4.PNG)

- **Prometheus Server URL**

```bash
http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local
```

![Grafana-5](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-5.PNG)

![Grafana-6](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-6.PNG)

![Grafana-7](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-7.PNG)

![Grafana-8](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-8.PNG)

![Grafana-9](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/grafana-9.PNG)


### E) Install Loki.

```bash
# Add the Grafana Helm chart repository to the local Helm configuration
helm repo add grafana https://grafana.github.io/helm-charts
```

```bash
# Create a values.yaml file with custom configurations for the Loki Helm chart.
cat <<EOF> values.yaml
loki:
  image:
    tag: 2.9.7
  persistence:
    enabled: true
    storageClassName: "nfs-client"
    size: 50Gi
EOF
```

```bash
# Install the Loki stack using the Helm chart from the Grafana repository
helm install loki grafana/loki-stack --namespace monitoring -f values.yaml
```

```bash
# List all pods in the 'monitoring' namespace to verify the deployment
# The output shows the status of pods, including Loki and Promtail instances
$ kubectl get pod -n monitoring
NAME                                                            READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0          2/2     Running   0          42m
grafana-6655c7ff45-cpl7v                                        1/1     Running   0          39m
loki-0                                                          1/1     Running   0          3m1s # <-
loki-promtail-89mj7                                             1/1     Running   0          3m1s # <-
loki-promtail-dxlrc                                             1/1     Running   0          3m1s # <-
loki-promtail-n6gzr                                             1/1     Running   0          3m1s # <-
prometheus-kube-prometheus-blackbox-exporter-559b9479ff-vpvmh   1/1     Running   0          42m
prometheus-kube-prometheus-operator-5f9dccf6bc-jlnjf            1/1     Running   0          42m
prometheus-kube-state-metrics-74cf974698-zlkrc                  1/1     Running   0          42m
prometheus-node-exporter-56rmf                                  1/1     Running   0          42m
prometheus-node-exporter-kdpcr                                  1/1     Running   0          42m
prometheus-prometheus-kube-prometheus-prometheus-0              2/2     Running   0          42m
```

```bash
kubectl port-forward -n monitoring service/loki --address 0.0.0.0 3100:3100 &
```

- **`Home > Connections > Data sources > Add data source`**

![loki-1](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-1.PNG)

```bash
# Loki Server URL
http://loki.monitoring.svc.cluster.local:3100
```

![loki-2](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-2.PNG)

![loki-3](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-3.PNG)

![loki-4](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-4.PNG)

![loki-5](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-5.PNG)

![loki-6](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-6.PNG)

![loki-7](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-7.PNG)

![loki-8](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-8.PNG)

![loki-9](https://github.com/revenge1005/k8s-cluster-setup/blob/main/07.%20Monitoring/img/loki-9.PNG)