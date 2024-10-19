Let's implement a microservices architecture example using Kubernetes probes, specifically focusing on a payment service. This service will have liveness and readiness probes to ensure it is healthy and capable of processing transactions.

### Scenario Overview

We'll create two microservices:
1. **Payment Service**: Simulates processing payments.
2. **Database Service**: Simulates a simple database connection for the payment service.

### Step 1: Create the Payment Service

1. **Set Up the Directory**:
   ```bash
   mkdir payment-service
   cd payment-service
   ```

2. **Create the Application Code** (`app.py`):
   ```python
   from flask import Flask, jsonify
   import time
   import random

   app = Flask(__name__)

   @app.route('/healthz')
   def health():
       return jsonify(status="healthy"), 200

   @app.route('/ready')
   def ready():
       if random.choice([True, False]):
           return jsonify(status="ready"), 200
       else:
           return jsonify(status="not ready"), 503

   @app.route('/pay', methods=['POST'])
   def pay():
       return jsonify(message="Payment processed"), 200

   if __name__ == "__main__":
       app.run(host='0.0.0.0', port=8080)
   ```

3. **Create a `requirements.txt` File**:
   ```
   Flask==2.0.1
   ```

4. **Create a Dockerfile**:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY . .

   RUN pip install --no-cache-dir -r requirements.txt
   EXPOSE 8080

   CMD ["python", "app.py"]
   ```

### Step 2: Build and Push the Docker Image

1. **Build the Docker Image**:
   ```bash
   docker build -t mypaymentservice:latest .
   ```

2. **Push the Image to a Registry**:
   ```bash
   docker tag mypaymentservice:latest <your-dockerhub-username>/mypaymentservice:latest
   docker push <your-dockerhub-username>/mypaymentservice:latest
   ```

### Step 3: Create the Database Service

1. **Set Up the Directory**:
   ```bash
   mkdir database-service
   cd database-service
   ```

2. **Create the Application Code** (`app.py`):
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/healthz')
   def health():
       return jsonify(status="healthy"), 200

   if __name__ == "__main__":
       app.run(host='0.0.0.0', port=8081)
   ```

3. **Create a `requirements.txt` File**:
   ```
   Flask==2.0.1
   ```

4. **Create a Dockerfile**:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY . .

   RUN pip install --no-cache-dir -r requirements.txt
   EXPOSE 8081

   CMD ["python", "app.py"]
   ```

### Step 4: Build and Push the Database Image

1. **Build the Docker Image**:
   ```bash
   docker build -t mydatabaseservice:latest .
   ```

2. **Push the Image to a Registry**:
   ```bash
   docker tag mydatabaseservice:latest <your-dockerhub-username>/mydatabaseservice:latest
   docker push <your-dockerhub-username>/mydatabaseservice:latest
   ```

### Step 5: Create Kubernetes Deployments

1. **Create a Deployment for the Payment Service** (`payment-deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: payment-service
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: payment
     template:
       metadata:
         labels:
           app: payment
       spec:
         containers:
         - name: payment-container
           image: <your-dockerhub-username>/mypaymentservice:latest
           ports:
           - containerPort: 8080
           livenessProbe:
             httpGet:
               path: /healthz
               port: 8080
             initialDelaySeconds: 15
             periodSeconds: 10
           readinessProbe:
             httpGet:
               path: /ready
               port: 8080
             initialDelaySeconds: 5
             periodSeconds: 10
   ```

2. **Create a Deployment for the Database Service** (`database-deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: database-service
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: database
     template:
       metadata:
         labels:
           app: database
       spec:
         containers:
         - name: database-container
           image: <your-dockerhub-username>/mydatabaseservice:latest
           ports:
           - containerPort: 8081
           livenessProbe:
             httpGet:
               path: /healthz
               port: 8081
             initialDelaySeconds: 15
             periodSeconds: 10
   ```

### Step 6: Apply the Deployments

1. **Apply the Database Service Deployment**:
   ```bash
   kubectl apply -f database-deployment.yaml
   ```

2. **Apply the Payment Service Deployment**:
   ```bash
   kubectl apply -f payment-deployment.yaml
   ```

### Step 7: Verify the Deployments

1. **Check the Pods**:
   ```bash
   kubectl get pods
   ```

   You should see both the payment and database service Pods running.

2. **Check the Logs** (optional):
   ```bash
   kubectl logs <pod-name>
   ```

### Step 8: Test the Probes

1. **Get the Payment Service Pod IP**:
   ```bash
   kubectl get pods -o wide
   ```

2. **Use `kubectl port-forward` to Access the Payment Service**:
   ```bash
   kubectl port-forward <payment-pod-name> 8080:8080
   ```

3. **Test the Endpoints**:
   - Check the health:
     ```bash
     curl http://localhost:8080/healthz
     ```
     You should see:
     ```
     {"status": "healthy"}
     ```

   - Check readiness:
     ```bash
     curl http://localhost:8080/ready
     ```
     You may get:
     ```
     {"status": "ready"} or {"status": "not ready"}
     ```

   - Simulate a payment:
     ```bash
     curl -X POST http://localhost:8080/pay
     ```
     You should see:
     ```
     {"message": "Payment processed"}
     ```

### Step 9: Clean Up

1. **Delete the Deployments**:
   ```bash
   kubectl delete -f payment-deployment.yaml
   kubectl delete -f database-deployment.yaml
   ```

### Conclusion

In this hands-on implementation, we:
- Built a payment service and a simple database service using Flask.
- Deployed both services to Kubernetes with liveness and readiness probes.
- Tested the probes to ensure the services are functioning correctly.

This setup helps maintain the health and reliability of the microservices architecture. If you have any questions or need further assistance, feel free to ask!
