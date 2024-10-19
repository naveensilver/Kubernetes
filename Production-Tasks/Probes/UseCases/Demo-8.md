Let's implement an end-to-end example of using readiness probes in a Continuous Deployment (CD) pipeline in Kubernetes. This scenario will demonstrate how to ensure that new deployments do not affect the availability of the application by checking their health before routing traffic to them.

### Scenario Overview

In this example, we will:
- Create a simple web application.
- Set up a Kubernetes deployment with a readiness probe.
- Simulate a deployment pipeline that checks the health of a new version of the application before rolling it out.

### Step 1: Create the Web Application

1. **Create the Web Application**:
   Create a new directory for your application and navigate into it:
   ```bash
   mkdir cd-pipeline-app
   cd cd-pipeline-app
   ```

   Create a file named `app.py`:
   ```python
   from flask import Flask, jsonify
   import random
   import time

   app = Flask(__name__)

   @app.route('/health')
   def health():
       return jsonify(status="healthy"), 200

   @app.route('/api/data')
   def data():
       return jsonify(message="This is data from the web application."), 200

   if __name__ == "__main__":
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create a `requirements.txt` File**:
   ```
   Flask==2.0.1
   ```

3. **Create a Dockerfile** for the application:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY . .

   RUN pip install --no-cache-dir -r requirements.txt
   EXPOSE 5000

   CMD ["python", "app.py"]
   ```

4. **Build and Push the Docker Image**:
   ```bash
   docker build -t cd-pipeline-app:v1 .
   docker tag cd-pipeline-app:v1 <your-dockerhub-username>/cd-pipeline-app:v1
   docker push <your-dockerhub-username>/cd-pipeline-app:v1
   ```

### Step 2: Create the Deployment and Service

1. **Create a Deployment Manifest**: Create a file named `app-deployment.yaml`.

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: cd-pipeline-app
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: cd-pipeline-app
     template:
       metadata:
         labels:
           app: cd-pipeline-app
       spec:
         containers:
         - name: cd-pipeline-app
           image: <your-dockerhub-username>/cd-pipeline-app:v1
           ports:
           - containerPort: 5000
           readinessProbe:
             httpGet:
               path: /health
               port: 5000
             initialDelaySeconds: 5
             periodSeconds: 5
   ```

2. **Create a Service Manifest**: Create a file named `app-service.yaml`.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: cd-pipeline-app
   spec:
     type: LoadBalancer
     ports:
       - port: 80
         targetPort: 5000
     selector:
       app: cd-pipeline-app
   ```

### Step 3: Apply the Configurations

1. **Deploy the Application**:
   ```bash
   kubectl apply -f app-deployment.yaml
   kubectl apply -f app-service.yaml
   ```

### Step 4: Verify the Deployment

1. **Check the Pods**:
   ```bash
   kubectl get pods
   ```

   You should see multiple replicas of your web application running.

2. **Check the Service**:
   ```bash
   kubectl get services
   ```

   Note the external IP assigned to the service.

### Step 5: Test the Application

1. **Access the Application**:
   Use the external IP to test the application:
   ```bash
   curl http://<external-ip>/api/data
   ```

### Step 6: Simulate a New Deployment

1. **Update the Application**:
   Modify `app.py` to change the response message or status, indicating a new version (e.g., add a version endpoint):
   ```python
   @app.route('/version')
   def version():
       return jsonify(version="v1.1"), 200
   ```

2. **Rebuild and Push the New Docker Image**:
   ```bash
   docker build -t cd-pipeline-app:v2 .
   docker tag cd-pipeline-app:v2 <your-dockerhub-username>/cd-pipeline-app:v2
   docker push <your-dockerhub-username>/cd-pipeline-app:v2
   ```

3. **Update the Deployment**:
   Modify the deployment manifest to use the new image version:
   ```yaml
   spec:
     template:
       spec:
         containers:
         - name: cd-pipeline-app
           image: <your-dockerhub-username>/cd-pipeline-app:v2
   ```

   Apply the changes:
   ```bash
   kubectl apply -f app-deployment.yaml
   ```

### Step 7: Monitor the Deployment

1. **Check the Pod Status**:
   Monitor the Pods as they update:
   ```bash
   kubectl get pods -w
   ```

2. **Verify Readiness Probes**:
   As the new Pods are created, Kubernetes will use the readiness probes to determine when they are healthy. Until the readiness check passes, traffic will not be routed to the new Pods.

3. **Access the Updated Application**:
   After a successful deployment, you can check the new version:
   ```bash
   curl http://<external-ip>/version
   ```

### Step 8: Clean Up

1. **Delete All Resources**:
   ```bash
   kubectl delete -f app-deployment.yaml
   kubectl delete -f app-service.yaml
   ```

### Explanation of Key Components

- **Readiness Probe**: Ensures that only healthy Pods receive traffic during and after deployment. This prevents downtime by verifying that the new version is functioning correctly before making it live.
  
- **Load Balancer Service**: Distributes incoming traffic to the available Pods based on their health status.

- **Continuous Deployment Process**: By leveraging readiness probes, the deployment pipeline can ensure that new versions are healthy before routing traffic to them. This approach minimizes the risk of deploying faulty applications and maintains high availability.

### Conclusion

In this implementation, we:
- Created a simple web application with health checks.
- Deployed it in Kubernetes with readiness probes.
- Simulated a new version deployment and monitored the readiness of the new Pods.

This approach demonstrates how to effectively integrate readiness probes into a continuous deployment pipeline, ensuring that application availability is maintained throughout the deployment process. If you have further questions or need additional details, feel free to ask!
