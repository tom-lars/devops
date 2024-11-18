# Ingress For K8s
### Installing HELM and creating ingress controller

First we need to install helm in our machine that we’ll use to push commands on control plane

```bash
curl https://raw.githubusercontent.com/helm/helm/master/static/get-helm-3 | bash
```

Then we will use helm to add a repo for nginx ingress controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

To check the repo with `helm repo list`

Then we will create a namespace for ingress controller

```bash
kubectl create ns ingress-nginx
```

next, we will install ingress controller with this command

```bash
helm install <ingress-controller-name> -n ingress-nginx ingress-nginx/ingress-nginx
```

### Creating deployment, service and ingress resource

Deployment and service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
  labels:
    run: nginx
spec:
    replicas: 2
    selector:
      matchLabels:
        run: nginx
    template:
      metadata:
        labels:
          run: nginx
      spec:
        containers:
        - name: nginx
          image: nginx   
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    run: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Ingress resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```

### DNS configuration

- Get an domain name and put it in `host` and replace `example.com`.
- Use `kubectl get svc -n ingress-nginx` to get the IP of the LoadBalancer.
    
    > If you are using EKS then the LoadBalancer will have DNS name, to get the IP from that DNS name use `host -v <DNS name>`.
    > 
- Then go to your DNS provider and in DNS configuration add an `A record` and map you LoadBalancer IP to a Subdomain.

## For Multi-Path routing and Muti-SubDomain routing

Deployment file

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue
  labels:
    run: blue
spec:
  replicas: 2
  selector:
    matchLabels:
      run: blue
  template:
    metadata:
      labels:
        run: blue
    spec:
      containers:
      - name: nginx
        image: siddocker467/blue:latest
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: red
  labels:
    run: red
spec:
  replicas: 2
  selector:
    matchLabels:
      run: red
  template:
    metadata:
      labels:
        run: red
    spec:
      containers:
      - name: nginx
        image: siddocker467/red:latest
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static
  labels:
    run: static
spec:
  replicas: 2
  selector:
    matchLabels:
      run: static
  template:
    metadata:
      labels:
        run: static
    spec:
      containers:
      - name: nginx
        image: siddocker467/static:latest

```

Service file

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-svc
spec:
  selector:
    run: blue
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: red-svc
spec:
  selector:
    run: red
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: static-svc
spec:
  selector:
    run: static
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Manifest for Multi-Path routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-1
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: path.devopscity.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: static-svc
            port:
              number: 80
      - path: /blue
        pathType: Prefix
        backend:
          service:
            name: blue-svc
            port:
              number: 80
      - path: /red
        pathType: Prefix
        backend:
          service:
            name: red-svc
            port:
              number: 80
```




https://github.com/user-attachments/assets/4e923dff-81c4-4242-84a6-595457da380a




Manifest for Multi-subdomain Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-2
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: static.devopscity.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: static-svc
            port:
              number: 80
  - host: blue.devopscity.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blue-svc
            port:
              number: 80
  - host: red.devopscity.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: red-svc
            port:
              number: 80
```

> when we use multi-subdomain routing, we need to add A records in our DNS for as many subdomain as we need.
>




https://github.com/user-attachments/assets/17ec621a-9b4e-4548-8b32-132f3a886c86



