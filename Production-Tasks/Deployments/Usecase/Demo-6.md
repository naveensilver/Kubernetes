Let’s implement an end-to-end example of a Rolling Update strategy using Kubernetes. This approach allows you to update applications with zero downtime by gradually replacing old Pods with new ones, ensuring that the desired number of replicas is always available. We’ll use a simple web application to demonstrate this.

### Step 1: Create the Web Application

We will use a basic Flask web application that will show its version in the response.

#### Create the Application

1. **Create Version 1 of the Application (`app_v1.py`)**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def hello():
       return jsonify({"version": "v1", "message": "Welcome to Version 1!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create Version 2 of the Application (`app_v2.py`)**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def hello():
       return jsonify({"version": "v2", "message": "Welcome to Version 2!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

### Step 2: Build and Push Docker Images

1. **Build and Push Version 1**:
   ```bash
   # For Version 1
   docker build -t <your-dockerhub-username>/rolling-update-v1:latest -f Dockerfile.v1 .
   docker push <your-dockerhub-username>/rolling-update-v1:latest
   ```

2. **Build and Push Version 2**:
   ```bash
   # For Version 2
   docker build -t <your-dockerhub-username>/rolling-update-v2:latest -f Dockerfile.v2 .
   docker push <your-dockerhub-username>/rolling-update-v2:latest
   ```

### Step 3: Deploy the Initial Version

#### 1. Create Deployment YAML File for Version 1 (`deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rolling-update-demo
  template:
    metadata:
      labels:
        app: rolling-update-demo
    spec:
      containers:
      - name: rolling-update-demo
        image: <your-dockerhub-username>/rolling-update-v1:latest
        ports:
        - containerPort: 5000
```

#### 2. Deploy Version 1
```bash
kubectl apply -f deployment.yaml
```

### Step 4: Expose the Application

1. **Create a Service YAML File (`service.yaml`)**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rolling-update-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: rolling-update-demo
```

2. **Apply the Service**:
```bash
kubectl apply -f service.yaml
```

### Step 5: Test the Initial Deployment

1. **Get the External IP of the Service**:
```bash
kubectl get services
```

2. **Access the Application**:
Visit the external IP address in your browser. You should see the response indicating "Welcome to Version 1!".

### Step 6: Perform a Rolling Update

#### 1. Update the Deployment YAML to Use Version 2
Modify `deployment.yaml` to point to Version 2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rolling-update-demo
  template:
    metadata:
      labels:
        app: rolling-update-demo
    spec:
      containers:
      - name: rolling-update-demo
        image: <your-dockerhub-username>/rolling-update-v2:latest  # Change to version 2
        ports:
        - containerPort: 5000
```

#### 2. Apply the Updated Deployment
```bash
kubectl apply -f deployment.yaml
```

### Step 7: Monitor the Rolling Update

1. **Check the Status of the Pods**:
```bash
kubectl get pods
```

You will see that Kubernetes gradually updates the Pods one by one. You can use the following command to watch the rollout:
```bash
kubectl rollout status deployment/rolling-update-demo
```

### Step 8: Test the Updated Deployment

1. **Access the Application Again**:
Visit the external IP address in your browser. You should now see the response indicating "Welcome to Version 2!".

### Step 9: Rollback (If Necessary)

If you need to rollback to the previous version, you can use the following command:

```bash
kubectl rollout undo deployment/rolling-update-demo
```

### Step 10: Clean Up

1. **Delete the Services and Deployments**:
```bash
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml
```

### Conclusion

In this end-to-end implementation, we:
- Created two versions of a web application.
- Deployed the initial version and exposed it using a Service.
- Performed a rolling update to transition to the new version without downtime.
- Monitored the rollout and tested the application post-update.

This Rolling Update strategy ensures that your application remains available during updates, making it an effective approach for maintaining high availability in production environments. If you have further questions or need assistance, feel free to ask!
