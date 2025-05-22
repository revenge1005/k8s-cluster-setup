# 0. 선행 작업

- [**1. Installing Kubernetes with Docker Engine Runtime**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)

or

- [**2. Installing Kubernetes with containerd Runtime**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)

<br>

# 1. Enable Dashboard

<BR>

### A) Install Dashboard 

```bash
# Install Helm.
{
    curl -O https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    bash ./get-helm-3
    helm version
}
```

```bash
# Install Dashboard 
{
    helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
    helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
}
```

```bash
$ kubectl get pods -n kubernetes-dashboard
NAME                                                    READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-api-85cf795569-q7pzn               1/1     Running   0          54s
kubernetes-dashboard-auth-698997cc4d-qzk6x              1/1     Running   0          54s
kubernetes-dashboard-kong-79867c9c48-p84b9              1/1     Running   0          54s
kubernetes-dashboard-metrics-scraper-76df4956c4-2xm5d   1/1     Running   0          54s
kubernetes-dashboard-web-56df7655d9-8gsjz               1/1     Running   0          54s
```

<BR>

### B) 	Add an account for Dashboard management.

```bash
# admin-user 서비스 계정 생성
kubectl create serviceaccount admin-user -n kubernetes-dashboard

cat <<EOF > rbac.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF 

kubectl apply -f rbac.yml
```

```bash
# get security token of the account above
$ kubectl -n kubernetes-dashboard create token admin-user
eyJhbGciOiJSUz.....
```

```bash
# to access to dashboard, set port-forwarding
$ kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard-kong-proxy --address 0.0.0.0 8443:443
Forwarding from 0.0.0.0:8443 -> 8443
```

<BR>

### C) Access to the URL below on a client computer in your local network.

- `https://(Control-plan Node IP address):8443/`

or

- If you have configured multi nodes, `https://(Manager(porxy,admin) Node IP address):8443/`

![03-2-1](https://github.com/revenge1005/k8s-cluster-setup/blob/main/03.%20multi-node_Dashboard/03-4.%20Enable%20Dashboard/03-2-1.PNG)

![03-2-2](https://github.com/revenge1005/k8s-cluster-setup/blob/main/03.%20multi-node_Dashboard/03-4.%20Enable%20Dashboard/03-2-2.PNG)