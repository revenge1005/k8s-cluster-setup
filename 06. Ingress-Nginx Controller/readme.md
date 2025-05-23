# 0. 선행 작업

- [**04. Installing MetalLB**](https://github.com/revenge1005/k8s-cluster-setup/tree/main/04.%20MetalLB)

<br>

1. Install Ingress-Nginx Controller


### A) Install Ingress-Nginx Controller

- https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

- https://github.com/kubernetes/ingress-nginx

```bash
# Select only one and install it.

# 1. Helm
{
    # Install Helm
    curl -O https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    bash ./get-helm-3
    helm upgrade --install ingress-nginx ingress-nginx \
        --repo https://kubernetes.github.io/ingress-nginx \
        --namespace ingress-nginx --create-namespace
}

# or

# 2. If you don't have Helm
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.2/deploy/static/provider/cloud/deploy.yaml
```

```bash
$ kubectl get all -n ingress-nginx
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-n4gjz        0/1     Completed   0          10m
pod/ingress-nginx-admission-patch-hf7lr         0/1     Completed   1          10m
pod/ingress-nginx-controller-7989649c5c-jkm9t   1/1     Running     0          10m

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.104.111.31   192.168.219.191   80:31991/TCP,443:31031/TCP   10m
service/ingress-nginx-controller-admission   ClusterIP      10.98.153.200   <none>            443/TCP                      10m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           10m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-7989649c5c   1         1         1       10m

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           9s         10m
job.batch/ingress-nginx-admission-patch    Complete   1/1           9s         10m
```

### B) Deployment for Test

```bash
cat <<EOF > app01.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - image: "strm/helloworld-http"
          name: hello-world-container
          ports:
            - containerPort: 80
  selector:
    matchLabels:
      app: hello-world
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld-svc
spec:
  type: NodePort
  ports:
     -  port: 8080
        protocol: TCP
        targetPort: 80
        nodePort: 31445
  selector:
    app: hello-world
EOF

kubectl apply -f app01.yaml
```

```bash
cat <<EOF > app02.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx
          name:  nginx
          ports:
            - containerPort: 80
  selector:
    matchLabels:
      app: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
     -  port: 9080
        targetPort: 80
EOF

kubectl apply -f app02.yaml
```

```bash
cat <<EOF > app03.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: liberty
    spec:
      containers:
        - image: openliberty/open-liberty:javaee8-ubi-min-amd64
          name:  open-liberty
          ports:
            - containerPort: 9080
              name: httpport
  selector:
    matchLabels:
      app: liberty
---
apiVersion: v1
kind: Service
metadata:
  name: java-svc
spec:
  selector:
    app: liberty
  ports:
     -  port: 9080
        targetPort: 9080
EOF

kubectl apply -f app03.yaml
```

### C) Create Ingress Manifest

```bash
cat <<EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: abc.sample.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: helloworld-svc
            port:
              number: 8080
      - path: /apl2
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 9080
  - host: xyz.sample.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-svc
            port:
              number:  9080
EOF

kubectl apply -f ingress.yaml
```

### D) Check

```bash
$ kubectl get all,ingress
NAME                                         READY   STATUS    RESTARTS   AGE
pod/helloworld-deployment-847c499fb8-v56mk   1/1     Running   0          2m26s
pod/ingress-nginx-demo-d9b5f5b46-vxqrx       1/1     Running   0          4m12s
pod/java-deployment-88d98b466-bsvwf          1/1     Running   0          2m20s
pod/nginx-deployment-86c57bc6b8-9whjl        1/1     Running   0          2m23s
pod/nginx-deployment-86c57bc6b8-dbxpl        1/1     Running   0          2m23s
pod/nginx-deployment-86c57bc6b8-s9f56        1/1     Running   0          2m23s

NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                         AGE
service/helloworld-svc   NodePort       10.102.124.193   <none>            8080:31445/TCP                  2m26s
service/java-svc         ClusterIP      10.103.190.215   <none>            9080/TCP                        2m20s
service/kubernetes       ClusterIP      10.96.0.1        <none>            443/TCP                         29m
service/nginx-svc        ClusterIP      10.99.15.42      <none>            9080/TCP                        2m23s

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helloworld-deployment   1/1     1            1           2m26s
deployment.apps/java-deployment         1/1     1            1           2m20s
deployment.apps/nginx-deployment        3/3     3            3           2m23s

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/helloworld-deployment-847c499fb8   1         1         1       2m26s
replicaset.apps/java-deployment-88d98b466          1         1         1       2m20s
replicaset.apps/nginx-deployment-86c57bc6b8        3         3         3       2m23s

NAME                                      CLASS    HOSTS                           ADDRESS           PORTS   AGE
ingress.networking.k8s.io/hello-ingress   <none>   abc.sample.com,xyz.sample.com   192.168.219.191   80      16s


$ kubectl -n ingress-nginx get service
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.104.111.31   192.168.219.191   80:31991/TCP,443:31031/TCP   21m
ingress-nginx-controller-admission   ClusterIP      10.98.153.200   <none>            443/TCP                      21m


$ kubectl describe ingress
Name:             hello-ingress
Labels:           <none>
Namespace:        default
Address:          192.168.219.191
Ingress Class:    <none>
Default backend:  <default>
Rules:
  Host            Path  Backends
  ----            ----  --------
  abc.sample.com
                  /       helloworld-svc:8080 (10.32.0.3:80)
                  /apl2   nginx-svc:9080 (10.36.0.4:80,10.32.0.4:80,10.32.0.5:80)
  xyz.sample.com
                  /   java-svc:9080 (10.36.0.5:9080)
Annotations:      kubernetes.io/ingress.class: nginx
                  nginx.ingress.kubernetes.io/rewrite-target: /
Events:           <none>
```

- windows - C:\Windows\System32\drivers\etc\hosts
- Linux - /etc/hosts

```bash
# hosts 파일에 아래 내용 추가
192.168.219.190  abc.sample.com xyz.sample.com
```

![1.gif](https://github.com/revenge1005/k8s-cluster-setup/blob/main/06.%20Ingress-Nginx%20Controller/1.gif)