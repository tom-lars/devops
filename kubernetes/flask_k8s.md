# Deploying flask application using ingress

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

### api.config.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  redis.host: "back-api"

```

### api.deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front-api
  template:
    metadata:
      labels:
        app: front-api
    spec:
      containers:
        - name: front-api
          image: siddocker467/flask01:latest
          args: ["gunicorn", "-w 1", "app:app", "-b 0.0.0.0:5000"]
          ports:
            - containerPort: 5000
          env:
            - name: REDIS_HOST
              valueFrom:
                configMapKeyRef:
                  name: api-config
                  key: redis.host
          resources:
            requests:
              cpu: "100m"        # Request 100 millicores of CPU
              memory: "128Mi"    # Request 128 MiB of memory
            limits:
              cpu: "500m"        # Limit to 500 millicores of CPU
              memory: "512Mi"    # Limit to 512 MiB of memory
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: front-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: front-api
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # Target 50% average CPU utilization

```

### api.service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-api
spec:
  type: ClusterIP
  selector:
    app: front-api
  ports:
    - protocol: TCP
      port: 5000 # port exposed internally in the cluster
      targetPort: 5000 # the container port to send requests to

```

### Applying taint on a node

```bash
kubectl taint nodes node1 purpose=redis:NoSchedule
```

### redis.deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: back-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: back-api
  template:
    metadata:
      labels:
        app: back-api
    spec:
      tolerations:
        - key: "purpose"
          operator: "Equal"
          value: "redis"
          effect: "NoSchedule"
      containers:
        - name: back-api
          image: redis
          ports:
            - containerPort: 6379
              name: redis
          resources:
            requests:
              cpu: "100m"         # Request 100 millicores of CPU
              memory: "128Mi"     # Request 128 MiB of memory
            limits:
              cpu: "500m"         # Limit to 500 millicores of CPU
              memory: "512Mi"     # Limit to 512 MiB of memory

---

# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: back-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: back-api
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # Target 50% average CPU utilization

```

### redis.service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: back-api
spec:
  type: ClusterIP
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    app: back-api
```

### ingress.resource.flask.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource
spec:
  ingressClassName: nginx
  rules:
  - host: flask.devopscity.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: front-api
            port:
              number: 5000

```
