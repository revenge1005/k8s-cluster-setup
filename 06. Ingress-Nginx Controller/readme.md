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
```