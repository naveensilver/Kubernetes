Let's implement the use case for **Monitoring and Alerting Configuration** using Kubernetes ConfigMaps. This example will demonstrate how to configure Prometheus to scrape metrics from a sample application using configurations stored in a ConfigMap.

### Use Case: Monitoring and Alerting Configuration

In this scenario:
- We'll use a **ConfigMap** to store Prometheus scrape configurations.
- We'll set up a simple Flask application that exposes metrics using the `prometheus_client` library.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-monitoring-demo
   cd k8s-monitoring-demo
   ```

### Step 2: Create the Flask Application with Metrics

1. **Create the application code**:
   - **Create a file named `app.py`**:
   ```python
   from flask import Flask, jsonify
   from prometheus_client import start_http_server, Counter
   import random
   import time

   app = Flask(__name__)

   REQUEST_COUNT = Counter('request_count', 'Total Request Count')

   @app.route('/')
   def index():
       REQUEST_COUNT.inc()
       return jsonify({"message": "Hello, monitoring world!"})

   if __name__ == '__main__':
       start_http_server(5001)  # Expose metrics on port 5001
       app.run(host='0.0.0.0', port=5000)
   ```

   - **Explanation**: This application serves a simple JSON message and counts the number of requests. It exposes Prometheus metrics on port 5001.

2. **Create a `requirements.txt` file**:
   ```
   Flask==2.0.1
   prometheus_client==0.11.0
   ```

3. **Create a Dockerfile**:
   ```dockerfile
   FROM python:3.9

   WORKDIR /usr/src/app
   COPY requirements.txt ./
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .

   CMD ["python", "app.py"]
   ```

### Step 3: Build the Docker Image

1. **Build the Docker image**:
   ```bash
   docker build -t k8s-monitoring-demo .
   ```

### Step 4: Create a ConfigMap for Prometheus Configuration

1. **Create a ConfigMap to store Prometheus scrape configuration**:
   Create a file named `prometheus-config.yaml`:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: prometheus-config
   data:
     prometheus.yml: |
       global:
         scrape_interval: 15s
       scrape_configs:
         - job_name: 'flask-app'
           static_configs:
             - targets: ['flask-app-service:5001']
   ```

2. **Apply the ConfigMap**:
   ```bash
   kubectl apply -f prometheus-config.yaml
   ```

3. **Verify the ConfigMap**:
   ```bash
   kubectl get configmaps
   ```

### Step 5: Create a Deployment for the Flask Application

1. **Create a deployment manifest named `flask-deployment.yaml`**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-app
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: flask-app
     template:
       metadata:
         labels:
           app: flask-app
       spec:
         containers:
         - name: app
           image: k8s-monitoring-demo
           ports:
           - containerPort: 5000
           - containerPort: 5001
   ```

2. **Apply the deployment**:
   ```bash
   kubectl apply -f flask-deployment.yaml
   ```

### Step 6: Create a Service for the Flask Application

1. **Create a service manifest named `flask-service.yaml`**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-app-service
   spec:
     selector:
       app: flask-app
     ports:
       - protocol: TCP
         port: 5000
         targetPort: 5000
       - protocol: TCP
         port: 5001
         targetPort: 5001
   ```

2. **Apply the service configuration**:
   ```bash
   kubectl apply -f flask-service.yaml
   ```

### Step 7: Deploy Prometheus

1. **Create a deployment manifest for Prometheus named `prometheus-deployment.yaml`**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: prometheus
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: prometheus
     template:
       metadata:
         labels:
           app: prometheus
       spec:
         containers:
         - name: prometheus
           image: prom/prometheus
           args:
             - '--config.file=/etc/prometheus/prometheus.yml'
           ports:
           - containerPort: 9090
           volumeMounts:
           - name: prometheus-config-volume
             mountPath: /etc/prometheus/
         volumes:
         - name: prometheus-config-volume
           configMap:
             name: prometheus-config
   ```

2. **Apply the Prometheus deployment**:
   ```bash
   kubectl apply -f prometheus-deployment.yaml
   ```

### Step 8: Expose Prometheus

1. **Create a service manifest for Prometheus named `prometheus-service.yaml`**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: prometheus-service
   spec:
     type: NodePort
     ports:
       - port: 9090
         targetPort: 9090
         nodePort: 30002
     selector:
       app: prometheus
   ```

2. **Apply the service configuration**:
   ```bash
   kubectl apply -f prometheus-service.yaml
   ```

### Step 9: Access the Application and Prometheus

1. **Get your Node's IP address**:
   ```bash
   kubectl get nodes -o wide
   ```

2. **Open your browser and navigate to the Flask application**:
   ```
   http://<node-ip>:30000/
   ```
   Replace `<node-ip>` with the actual IP address you found. You should see a JSON response:
   ```json
   {
       "message": "Hello, monitoring world!"
   }
   ```

3. **Access Prometheus**:
   Open your browser and navigate to:
   ```
   http://<node-ip>:30002/
   ```
   You should see the Prometheus UI.

4. **Check Targets**:
   In the Prometheus UI, go to `Status` > `Targets`. You should see the `flask-app` target listed and its status as `UP`.

### Step 10: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service prometheus-service
kubectl delete deployment prometheus
kubectl delete service flask-app-service
kubectl delete deployment flask-app
kubectl delete configmap prometheus-config
```

### Conclusion

In this implementation, we:
1. Created a Flask application that exposes metrics using Prometheus.
2. Set up a ConfigMap to store Prometheus scrape configurations.
3. Deployed Prometheus to scrape metrics from our application.
4. Verified that Prometheus could access our application's metrics.

This use case demonstrates how to configure monitoring tools like Prometheus using ConfigMaps in a Kubernetes environment. If you have any further questions or need additional examples, feel free to ask!
