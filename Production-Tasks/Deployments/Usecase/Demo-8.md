Let’s go through an end-to-end implementation of deploying monitoring and logging solutions using **Prometheus** for metrics collection and **Grafana** for visualization in a Kubernetes cluster. This setup will allow you to monitor the health and performance of your applications in real-time.

### Step 1: Prerequisites

1. **Kubernetes Cluster**: Ensure you have a running Kubernetes cluster.
2. **kubectl**: Make sure `kubectl` is installed and configured to interact with your cluster.
3. **Helm**: Install Helm, a package manager for Kubernetes, to simplify the deployment process.

### Step 2: Install Prometheus and Grafana Using Helm

Using Helm, you can easily install Prometheus and Grafana by leveraging community-maintained charts.

#### 1. Add the Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### 2. Install Prometheus

```bash
helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace
```

This command does the following:
- Installs Prometheus using the Helm chart.
- Creates a namespace called `monitoring` to keep monitoring resources isolated.

#### 3. Install Grafana

```bash
helm install grafana grafana/grafana --namespace monitoring
```

This command installs Grafana in the same namespace.

### Step 3: Verify Deployments

1. **Check the Status of Pods**:

```bash
kubectl get pods -n monitoring
```

You should see Pods for both Prometheus and Grafana in a `Running` state.

2. **Get Services**:

```bash
kubectl get services -n monitoring
```

This will show you the services, including Grafana and Prometheus.

### Step 4: Access Grafana

1. **Port Forward to Access Grafana**:

```bash
kubectl port-forward svc/grafana 3000:80 -n monitoring
```

You can now access Grafana by going to `http://localhost:3000` in your web browser.

2. **Log in to Grafana**:

- The default username is `admin`.
- The default password is also `admin`. You will be prompted to change it on first login.

### Step 5: Configure Data Source

1. **Add Prometheus as a Data Source**:

- After logging into Grafana, go to **Configuration > Data Sources**.
- Click on **Add data source**, then select **Prometheus**.
- In the URL field, enter: `http://prometheus-server.monitoring.svc.cluster.local:80` (this is the default service endpoint).
- Click **Save & Test** to verify the connection.

### Step 6: Create a Dashboard

1. **Create a New Dashboard**:

- Go to **Create > Dashboard**.
- Click on **Add new panel**.
- In the **Query** section, you can enter Prometheus queries to visualize metrics. For example, to monitor CPU usage, you might use:
  ```plaintext
  sum(rate(container_cpu_usage_seconds_total[5m])) by (container_name)
  ```
- Customize the visualization type (e.g., Graph, Gauge) as needed.
- Click **Apply** to add the panel to your dashboard.

### Step 7: Monitor Application Health

1. **Deploy a Sample Application** (Optional):

To visualize real metrics, you can deploy a sample application, such as a simple web server. Here’s a quick deployment you can use:

**File: `sample-app.yaml`**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx
        ports:
        - containerPort: 80
```

2. **Deploy the Sample Application**:

```bash
kubectl apply -f sample-app.yaml
```

### Step 8: Cleanup Resources

1. **Delete Sample Application** (if deployed):

```bash
kubectl delete -f sample-app.yaml
```

2. **Uninstall Prometheus and Grafana**:

```bash
helm uninstall prometheus --namespace monitoring
helm uninstall grafana --namespace monitoring
kubectl delete namespace monitoring
```

### Conclusion

In this end-to-end implementation, we:
- Installed Prometheus and Grafana using Helm.
- Accessed Grafana and configured it to use Prometheus as a data source.
- Created a dashboard to visualize application metrics.

This monitoring setup enables teams to monitor application health and performance effectively, facilitating proactive management in production environments. If you have further questions or need assistance, feel free to ask!
