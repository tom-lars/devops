To install Prometheus and Grafana on Amazon EKS using Helm, follow these steps:

### Prerequisites:
1. **Helm** installed on your local machine or cloud environment.
2. **kubectl** installed and configured to access your EKS cluster.
3. **AWS CLI** installed and configured with the necessary permissions.

### Step 1: Set up Helm on EKS

If you haven’t already, initialize Helm on your EKS cluster by running:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Step 2: Install Prometheus using Helm

1. **Create a namespace** for Prometheus (optional but recommended):

```bash
kubectl create namespace monitoring
```

2. **Install Prometheus using Helm**:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
```

This command installs the Prometheus stack, which includes Prometheus, Alertmanager, and the required configurations.

### Step 3: Install Grafana using Helm

You can install Grafana directly as part of the `kube-prometheus-stack`, but if you want to install Grafana separately, follow these steps:

1. **Install Grafana**:

```bash
helm install grafana grafana/grafana --namespace monitoring
```

### Step 4: Access Grafana Dashboard

To access the Grafana dashboard:

1. **Port-forward** the Grafana service to your local machine:

```bash
kubectl port-forward service/grafana 3000:80 --namespace monitoring
```

2. Open a browser and go to `http://localhost:3000`. The default username and password are:

- **Username:** `admin`
- **Password:** `admin` (or you can get the password using `kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode`)

### Step 5: Set up Prometheus as a Data Source in Grafana

1. After logging into Grafana, navigate to **Configuration > Data Sources**.
2. Select **Prometheus** and set the **URL** to `http://prometheus-operated.monitoring.svc:9090` (default internal Prometheus URL).
3. Save and test the connection.

### Step 6: Create Dashboards in Grafana

1. Navigate to **Create > Dashboard** in Grafana.
2. Add panels and select the Prometheus data source to visualize metrics.

### Optional: Set up Persistent Storage (if needed)

If you want to persist Prometheus data, you can enable persistent volume claims by modifying the Helm chart values. Create a custom `values.yaml`:

```yaml
prometheus:
  server:
    persistentVolume:
      enabled: true
      size: 50Gi
```

Then install Prometheus with the custom values:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring -f values.yaml
```

That’s it! You now have Prometheus and Grafana running on EKS using Helm.
