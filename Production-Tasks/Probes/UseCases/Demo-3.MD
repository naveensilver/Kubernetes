Let’s implement an example of a long-running job using a machine learning model server in Kubernetes, focusing on startup probes to ensure the application is fully initialized before it starts accepting traffic.

### Scenario Overview

We'll create a simple Flask application that simulates loading a machine learning model. We will use a startup probe to ensure the application is ready to serve requests only after the model is fully loaded.

### Step 1: Create the Machine Learning Model Server

1. **Set Up the Directory**:
   ```bash
   mkdir ml-model-server
   cd ml-model-server
   ```

2. **Create the Application Code** (`app.py`):
   ```python
   from flask import Flask, jsonify
   import time

   app = Flask(__name__)

   model_loaded = False

   @app.before_first_request
   def load_model():
       global model_loaded
       # Simulating model loading time
       print("Loading model...")
       time.sleep(30)  # Simulate time taken to load model
       model_loaded = True
       print("Model loaded.")

   @app.route('/healthz')
   def health():
       return jsonify(status="healthy"), 200

   @app.route('/ready')
   def ready():
       if model_loaded:
           return jsonify(status="ready"), 200
       else:
           return jsonify(status="not ready"), 503

   @app.route('/predict', methods=['POST'])
   def predict():
       if not model_loaded:
           return jsonify(error="Model not loaded"), 503
       return jsonify(prediction="This is a mock prediction"), 200

   if __name__ == "__main__":
       app.run(host='0.0.0.0', port=5000)
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
   EXPOSE 5000

   CMD ["python", "app.py"]
   ```

### Step 2: Build and Push the Docker Image

1. **Build the Docker Image**:
   ```bash
   docker build -t mymlmodelserver:latest .
   ```

2. **Push the Image to a Registry**:
   ```bash
   docker tag mymlmodelserver:latest <your-dockerhub-username>/mymlmodelserver:latest
   docker push <your-dockerhub-username>/mymlmodelserver:latest
   ```

### Step 3: Create Kubernetes Deployment

1. **Create a Deployment Manifest** (`ml-model-server-deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ml-model-server
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: ml-model
     template:
       metadata:
         labels:
           app: ml-model
       spec:
         containers:
         - name: ml-model-container
           image: <your-dockerhub-username>/mymlmodelserver:latest
           ports:
           - containerPort: 5000
           startupProbe:
             httpGet:
               path: /ready
               port: 5000
             initialDelaySeconds: 5
             periodSeconds: 5
           livenessProbe:
             httpGet:
               path: /healthz
               port: 5000
             initialDelaySeconds: 30
             periodSeconds: 10
           readinessProbe:
             httpGet:
               path: /ready
               port: 5000
             initialDelaySeconds: 5
             periodSeconds: 5
   ```

### Step 4: Apply the Deployment

1. **Deploy the ML Model Server**:
   ```bash
   kubectl apply -f ml-model-server-deployment.yaml
   ```

### Step 5: Verify the Deployment

1. **Check the Pods**:
   ```bash
   kubectl get pods
   ```

   You should see the ML model server Pod running.

2. **Check the Logs**:
   ```bash
   kubectl logs <pod-name>
   ```

   You should see logs indicating that the model is loading and once it's done, that it’s loaded.

### Step 6: Test the Probes

1. **Get the Pod IP**:
   ```bash
   kubectl get pods -o wide
   ```

2. **Use `kubectl port-forward` to Access the ML Model Server**:
   ```bash
   kubectl port-forward <pod-name> 5000:5000
   ```

3. **Test Readiness Before Model Load**:
   Check if the model server is ready:
   ```bash
   curl http://localhost:5000/ready
   ```
   You should receive a 503 response initially since the model is still loading.

4. **Test Readiness After Model Load**:
   After about 30 seconds, check again:
   ```bash
   curl http://localhost:5000/ready
   ```
   You should receive a 200 response indicating that the model is now ready.

5. **Test the Prediction Endpoint**:
   Once the model is loaded, you can test the prediction endpoint:
   ```bash
   curl -X POST http://localhost:5000/predict
   ```
   You should receive a mock prediction response.

### Step 7: Clean Up

1. **Delete the Deployment**:
   ```bash
   kubectl delete -f ml-model-server-deployment.yaml
   ```

### Explanation of Key Components

- **Startup Probe**: Checks if the model server is ready to accept traffic only after the model is fully loaded. This prevents traffic from being sent to the application while it's still initializing.
- **Liveness Probe**: Ensures the application is running. If it fails, Kubernetes will restart the Pod.
- **Readiness Probe**: Confirms that the application can handle requests. If it fails, Kubernetes will stop routing traffic to the Pod.

### Conclusion

In this hands-on implementation, we:
- Built a machine learning model server using Flask.
- Deployed the server in Kubernetes with startup, liveness, and readiness probes.
- Tested the probes to ensure the application only accepts traffic once the model is fully loaded.

This setup is essential for managing long-running jobs effectively in a Kubernetes environment. If you have any questions or need further details, feel free to ask!
