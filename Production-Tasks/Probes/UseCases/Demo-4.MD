Let’s implement an end-to-end example of an API Gateway using **Kong** as our gateway, demonstrating how readiness probes can be used to ensure that upstream services are healthy before routing requests to them.

### Scenario Overview

In this example, we will set up a simple microservices architecture with:
- A **Kong API Gateway** to manage traffic.
- A simple **backend service** (like a Flask application) that Kong will route requests to.
- Readiness probes to ensure that Kong only routes traffic to healthy services.

### Step 1: Set Up the Backend Service

1. **Create the Backend Service** (`app.py`):
   Create a directory for your backend service and a file named `app.py`.

   ```bash
   mkdir backend-service
   cd backend-service
   ```

   In `app.py`, add the following code:

   ```python
   from flask import Flask, jsonify
   import time

   app = Flask(__name__)

   @app.route('/health')
   def health():
       return jsonify(status="healthy"), 200

   @app.route('/api/data')
   def data():
       return jsonify(message="This is data from the backend service."), 200

   if __name__ == "__main__":
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create a `requirements.txt` File**:
   ```
   Flask==2.0.1
   ```

3. **Create a Dockerfile** for the backend service:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY . .

   RUN pip install --no-cache-dir -r requirements.txt
   EXPOSE 5000

   CMD ["python", "app.py"]
   ```

4. **Build and Push the Backend Docker Image**:
   ```bash
   docker build -t backend-service:latest .
   docker tag backend-service:latest <your-dockerhub-username>/backend-service:latest
   docker push <your-dockerhub-username>/backend-service:latest
   ```

### Step 2: Set Up the Kong API Gateway

1. **Create a Deployment for Kong**: Create a file named `kong-deployment.yaml`.

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: kong
     labels:
       app: kong
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: kong
     template:
       metadata:
         labels:
           app: kong
       spec:
         containers:
         - name: kong
           image: kong:latest
           env:
           - name: KONG_DATABASE
             value: "off"  # Using DB-less mode
           - name: KONG_PROXY_LISTEN
             value: "0.0.0.0:8000"
           ports:
           - containerPort: 8000
   ```

2. **Create a Service for Kong**: Create a file named `kong-service.yaml`.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: kong
   spec:
     ports:
       - port: 8000
         targetPort: 8000
     selector:
       app: kong
   ```

### Step 3: Create the Backend Deployment

1. **Create a Deployment Manifest for the Backend Service**: Create a file named `backend-deployment.yaml`.

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: backend
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: backend
     template:
       metadata:
         labels:
           app: backend
       spec:
         containers:
         - name: backend
           image: <your-dockerhub-username>/backend-service:latest
           ports:
           - containerPort: 5000
           readinessProbe:
             httpGet:
               path: /health
               port: 5000
             initialDelaySeconds: 5
             periodSeconds: 10
   ```

2. **Create a Service for the Backend**: Create a file named `backend-service.yaml`.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: backend
   spec:
     ports:
       - port: 5000
         targetPort: 5000
     selector:
       app: backend
   ```

### Step 4: Apply All Configurations

1. **Deploy the Backend Service**:
   ```bash
   kubectl apply -f backend-deployment.yaml
   kubectl apply -f backend-service.yaml
   ```

2. **Deploy Kong**:
   ```bash
   kubectl apply -f kong-deployment.yaml
   kubectl apply -f kong-service.yaml
   ```

### Step 5: Configure Kong to Route to the Backend Service

1. **Set Up Kong Ingress**: Create a file named `kong-ingress.yaml`.

   ```yaml
   apiVersion: configuration.konghq.com/v1
   kind: KongIngress
   metadata:
     name: backend
   proxy:
     path: /api
     service: backend
   ```

2. **Apply the Kong Ingress**:
   ```bash
   kubectl apply -f kong-ingress.yaml
   ```

### Step 6: Verify the Deployment

1. **Check the Pods**:
   ```bash
   kubectl get pods
   ```

2. **Check the Logs** (optional):
   ```bash
   kubectl logs <kong-pod-name>
   ```

3. **Get the IP and Port for Kong**:
   Use port forwarding to access Kong:
   ```bash
   kubectl port-forward svc/kong 8000:8000
   ```

### Step 7: Test the API Gateway

1. **Check the Health of the Backend**:
   Use the Kong API gateway to route requests:
   ```bash
   curl http://localhost:8000/api/data
   ```

   If the backend is healthy, you should receive a valid response. If not, the readiness probe will ensure Kong does not route traffic to it.

2. **Simulate Backend Failure**:
   You can temporarily scale down the backend service:
   ```bash
   kubectl scale deployment backend --replicas=0
   ```

3. **Check Kong Routing**:
   Try to access the API again:
   ```bash
   curl http://localhost:8000/api/data
   ```

   You should get an error indicating that the service is unavailable, as Kong will not route traffic to the unhealthy backend.

### Step 8: Clean Up

1. **Delete All Resources**:
   ```bash
   kubectl delete -f backend-deployment.yaml
   kubectl delete -f backend-service.yaml
   kubectl delete -f kong-deployment.yaml
   kubectl delete -f kong-service.yaml
   kubectl delete -f kong-ingress.yaml
   ```

### Explanation of Key Components

- **Kong API Gateway**: Acts as the entry point for requests, routing them to the appropriate backend services.
- **Readiness Probe**: Ensures that the backend service is healthy before Kong routes traffic to it. If the backend service is down, Kong will stop sending requests to it.
- **Service**: Exposes the backend service and Kong to enable communication within the cluster and external access.

### Conclusion

In this implementation, we:
- Created a backend service and an API gateway using Kong.
- Configured readiness probes to ensure the gateway only routes traffic to healthy services.
- Tested the entire flow, including simulating a backend failure to see how Kong manages routing.

This setup is crucial for managing microservices architecture effectively in a Kubernetes environment. If you have further questions or need additional details, feel free to ask!
