# 0. 선행 작업

- [1. Installing Kubernetes with Docker Engine Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-01.%20Docker%20Engine)

or

- [2. Installing Kubernetes with containerd Runtime](https://github.com/revenge1005/k8s-cluster-setup/tree/main/02.%20Container%20runtime/02-02.%20containerd)

<br>

# 2. Installing MetalLB 

- [MetalLB](https://metallb.universe.tf/installation/)

<br>

### A) strict ARP mode 활성화

```bash
kubectl edit configmap -n kube-system kube-proxy
```

```bash
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true  # <- false에서 true로 변경
```

<br>

### B) Installation by manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

```bash
$ kubectl get all -n metallb-system
NAME                             READY   STATUS    RESTARTS   AGE
pod/controller-bb5f47665-m86g5   1/1     Running   0          2m42s
pod/speaker-kv867                1/1     Running   0          2m42s
pod/speaker-pdnmg                1/1     Running   0          2m42s
pod/speaker-vxdfj                1/1     Running   0          2m42s
pod/speaker-zhnr4                1/1     Running   0          2m42s

NAME                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/metallb-webhook-service   ClusterIP   10.111.91.89   <none>        443/TCP   2m42s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   4         4         4       4            4           kubernetes.io/os=linux   2m42s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           2m42s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-bb5f47665   1         1         1       2m42s
```

<br>

### C) IP Address Pool + L2Advertisement configuration

```bash
cat <<EOF > my-network.yaml 
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.219.190-192.168.219.199
  autoAssign: true		# autoAssign 옵션을 true 해두면 로드밸런서 타입으로 서비스를 생성하면 자동으로 위의 IP 대역에서 IP를 할당한다.

---

apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  namespace: metallb-system
  name: example
spec:
  ipAddressPools:
  - first-pool		# 앞서 정의한 IP-Pool 이름을 넣어준다.
  nodeSelectors:	# first-pool IP를 통해서 들어오는 노드 선택
  - matchLabels:
      kubernetes.io/hostname: k8s-worker01		
  - matchLabels:
      kubernetes.io/hostname: k8s-worker02
  - matchLabels:
      kubernetes.io/hostname: k8s-worker03

EOF


kubectl apply -f my-network.yaml
```

<br>

### D) Check the operation of 'Metal LB'

```bash
cat <<EOF > test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: almighty
  labels:
    app: almighty
spec:
  containers:
  - name: almighty
    image: docker.io/andrewloyolajeong/almighty:0.2.4

---

apiVersion: v1
kind: Service
metadata:
  name: almighty
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: almighty
  ports:
    - name: myweb
      protocol: TCP
      port: 8080
      targetPort: 8080
    - name: yourweb
      protocol: TCP
      port: 1080
      targetPort: 80
EOF


kubectl apply -f test.yaml
```

```bash
kubectl get all -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP          NODE           NOMINATED NODE   READINESS GATES
pod/almighty                           1/1     Running   0          6m18s   10.39.0.7   k8s-worker03   <none>           <none>

NAME                      TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                         AGE     SELECTOR
service/almighty          LoadBalancer   10.106.91.31   192.168.219.190   8080:31668/TCP,1080:30307/TCP   6m18s   app=almighty
service/kubernetes        ClusterIP      10.96.0.1      <none>            443/TCP                         109m    <none>
```

![metallb-01](https://github.com/revenge1005/k8s-cluster-setup/blob/main/03.%20MetalLB/metallb-01.PNG)

![metallb-02](https://github.com/revenge1005/k8s-cluster-setup/blob/main/03.%20MetalLB/metallb-02.PNG)