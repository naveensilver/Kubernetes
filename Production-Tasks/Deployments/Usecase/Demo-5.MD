Let's implement an end-to-end example of a Blue-Green Deployment strategy using Kubernetes. This approach minimizes downtime and risk during updates by maintaining two identical environments (blue and green). In this scenario, we'll:

1. Create two identical versions of a web application (blue and green).
2. Deploy both versions to a Kubernetes cluster.
3. Use a Service to manage traffic between the two environments.
4. Demonstrate switching traffic between the two versions.

### Step 1: Create the Web Application

We'll use the same simple Flask web application for both versions. You can either use the previous versions or create new ones. 

#### Create the Application

1. **Create Version A (`app.py`)**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def hello():
       return jsonify({"version": "Blue", "message": "Welcome to the Blue environment!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create Version B (`app.py`)**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def hello():
       return jsonify({"version": "Green", "message": "Welcome to the Green environment!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

### Step 2: Build and Push Docker Images

1. **Build and Push Blue Version**:
   ```bash
   docker build -t <your-dockerhub-username>/blue-env:latest .
   docker push <your-dockerhub-username>/blue-env:latest
   ```

2. **Build and Push Green Version**:
   ```bash
   docker build -t <your-dockerhub-username>/green-env:latest .
   docker push <your-dockerhub-username>/green-env:latest
   ```

### Step 3: Deploy Blue and Green Environments

#### 1. Create Deployment YAML Files

1. **For Blue Environment (`blue-deployment.yaml`)**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: blue-env
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: blue-env
     template:
       metadata:
         labels:
           app: blue-env
       spec:
         containers:
         - name: blue-env
           image: <your-dockerhub-username>/blue-env:latest
           ports:
           - containerPort: 5000
   ```

2. **For Green Environment (`green-deployment.yaml`)**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: green-env
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: green-env
     template:
       metadata:
         labels:
           app: green-env
       spec:
         containers:
         - name: green-env
           image: <your-dockerhub-username>/green-env:latest
           ports:
           - containerPort: 5000
   ```

#### 2. Deploy Each Environment

1. **Apply the Blue Deployment**:
   ```bash
   kubectl apply -f blue-deployment.yaml
   ```

2. **Apply the Green Deployment**:
   ```bash
   kubectl apply -f green-deployment.yaml
   ```

### Step 4: Expose the Applications

1. **Create a Service YAML File (`app-service.yaml`)**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: app-service
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: blue-env  # Initially point to the blue environment
   ```

2. **Apply the Service**:
   ```bash
   kubectl apply -f app-service.yaml
   ```

### Step 5: Test the Blue Environment

1. **Get the External IP of the Service**:
   ```bash
   kubectl get services
   ```

2. **Access the Application**:
   Visit the external IP address in your browser. You should see the message from the Blue environment.

### Step 6: Switch to Green Environment

#### 1. Update the Service to Point to Green

1. **Update the Service**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: app-service
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: green-env  # Switch to the green environment
   ```

2. **Apply the Updated Service**:
   ```bash
   kubectl apply -f app-service.yaml
   ```

#### 2. Test the Green Environment

1. **Access the Application Again**:
   Visit the same external IP address in your browser. You should now see the message from the Green environment.

### Step 7: Rollback to Blue (if Needed)

If you encounter issues with the Green environment, you can easily switch back to Blue by updating the Service selector again.

1. **Update the Service Back to Blue**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: app-service
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: blue-env  # Switch back to the blue environment
   ```

2. **Apply the Updated Service**:
   ```bash
   kubectl apply -f app-service.yaml
   ```

### Step 8: Clean Up

1. **Delete the Services and Deployments**:
   ```bash
   kubectl delete -f blue-deployment.yaml
   kubectl delete -f green-deployment.yaml
   kubectl delete -f app-service.yaml
   ```

### Conclusion

In this end-to-end implementation, we:
- Created two identical environments (blue and green) for a web application.
- Containerized both versions using Docker and deployed them to Kubernetes.
- Set up a Service to manage traffic between the two environments, allowing us to switch between them easily.

This Blue-Green Deployment strategy ensures minimal downtime and quick rollback capabilities, making it a robust approach for managing application updates. If you have any further questions or need assistance, feel free to ask!
