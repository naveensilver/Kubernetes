Let’s walk through an end-to-end implementation of Kubernetes probes, focusing on liveness and readiness checks with a hands-on example.

### Step-by-Step Implementation of Probes in Kubernetes

#### Scenario

We'll create a simple web application that has two endpoints:
- `/healthz`: for liveness checks
- `/ready`: for readiness checks

The application will be a basic Python Flask app.

### Step 1: Create the Application

1. **Create a Directory for Your Application**:
   ```bash
   mkdir flask-app
   cd flask-app
   ```

2. **Create a Python File** (`app.py`):
   ```python
   from flask import Flask

   app = Flask(__name__)

   @app.route('/healthz')
   def health():
       return "Healthy", 200

   @app.route('/ready')
   def ready():
       return "Ready", 200

   if __name__ == "__main__":
       app.run(host='0.0.0.0', port=8080)
   ```

3. **Create a `requirements.txt` File**:
   ```
   Flask==2.0.1
   ```

4. **Create a Dockerfile**:
   ```dockerfile
   # Use an official Python runtime as a parent image
   FROM python:3.9-slim

   # Set the working directory in the container
   WORKDIR /app

   # Copy the current directory contents into the container at /app
   COPY . .

   # Install any needed packages specified in requirements.txt
   RUN pip install --no-cache-dir -r requirements.txt

   # Make port 8080 available to the world outside this container
   EXPOSE 8080

   # Define environment variable
   ENV NAME World

   # Run app.py when the container launches
   CMD ["python", "app.py"]
   ```

### Step 2: Build and Push the Docker Image

1. **Build the Docker Image**:
   ```bash
   docker build -t myflaskapp:latest .
   ```

2. **Push the Image to a Registry** (e.g., Docker Hub):
   ```bash
   docker tag myflaskapp:latest <your-dockerhub-username>/myflaskapp:latest
   docker push <your-dockerhub-username>/myflaskapp:latest
   ```

### Step 3: Create the Kubernetes Deployment

1. **Create a Deployment Manifest** (`deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-app
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: flask
     template:
       metadata:
         labels:
           app: flask
       spec:
         containers:
         - name: flask-container
           image: <your-dockerhub-username>/myflaskapp:latest
           ports:
           - containerPort: 8080
           livenessProbe:
             httpGet:
               path: /healthz
               port: 8080
             initialDelaySeconds: 30
             periodSeconds: 10
           readinessProbe:
             httpGet:
               path: /ready
               port: 8080
             initialDelaySeconds: 5
             periodSeconds: 10
   ```

2. **Apply the Deployment**:
   ```bash
   kubectl apply -f deployment.yaml
   ```

### Step 4: Verify the Deployment

1. **Check the Pods**:
   ```bash
   kubectl get pods
   ```

   You should see two Pods running for the `flask-app`.

2. **Check the Logs** (optional):
   ```bash
   kubectl logs <pod-name>
   ```

### Step 5: Test the Probes

1. **Get the Pod IP**:
   ```bash
   kubectl get pods -o wide
   ```

2. **Use `kubectl port-forward` to Access the Application**:
   ```bash
   kubectl port-forward <pod-name> 8080:8080
   ```

3. **Test the Endpoints**:
   - Open another terminal and use `curl`:
     ```bash
     curl http://localhost:8080/healthz
     ```
     You should see:
     ```
     Healthy
     ```

   - Test readiness:
     ```bash
     curl http://localhost:8080/ready
     ```
     You should see:
     ```
     Ready
     ```

### Step 6: Simulate Failure for Liveness Probe

1. **Modify the `app.py`**:
   You can simulate a failure by temporarily changing the `/healthz` endpoint to return a 500 status:
   ```python
   @app.route('/healthz')
   def health():
       return "Unhealthy", 500  # Simulate failure
   ```

2. **Rebuild and Push the Image**:
   ```bash
   docker build -t myflaskapp:latest .
   docker push <your-dockerhub-username>/myflaskapp:latest
   ```

3. **Update the Deployment**:
   ```bash
   kubectl rollout restart deployment flask-app
   ```

4. **Check the Pods**:
   After a short delay, check the logs to see if Kubernetes restarted the container.

### Step 7: Clean Up

1. **Delete the Deployment**:
   ```bash
   kubectl delete deployment flask-app
   ```

### Conclusion

In this hands-on implementation, we:
- Built a simple Flask application with health and readiness endpoints.
- Deployed the application to Kubernetes with liveness and readiness probes.
- Tested the application and simulated failure to see how Kubernetes reacts.

This approach helps maintain application availability and reliability in production. If you have any questions or need further assistance, feel free to ask!
