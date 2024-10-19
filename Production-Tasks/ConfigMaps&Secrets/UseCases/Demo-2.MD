Let's go through a detailed end-to-end implementation of the second use case: **Feature Flags** using ConfigMaps and Secrets in Kubernetes. This will allow us to enable or disable features in our application dynamically.

### Use Case: Feature Flags

In this scenario:
- We'll use a **ConfigMap** to store feature flags that control whether certain features are enabled or disabled in a Python Flask application.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-feature-flags-demo
   cd k8s-feature-flags-demo
   ```

### Step 2: Create the Python Application

1. **Create the application code**:
   - **Create a file named `app.py`**:
   ```python
   from flask import Flask
   import os

   app = Flask(__name__)

   @app.route('/')
   def index():
       feature_a = os.environ.get('FEATURE_A', 'false').lower() == 'true'
       feature_b = os.environ.get('FEATURE_B', 'false').lower() == 'true'
       
       response = "Features:\n"
       response += f"Feature A is {'enabled' if feature_a else 'disabled'}\n"
       response += f"Feature B is {'enabled' if feature_b else 'disabled'}\n"
       return response

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

   - **Explanation**: This application checks the status of two features (A and B) from environment variables and returns their status.

2. **Create a `requirements.txt` file**:
   ```
   Flask==2.0.1
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
   docker build -t k8s-feature-flags-demo .
   ```

### Step 4: Create a ConfigMap for Feature Flags

1. **Create a ConfigMap for the feature flags**:
   ```bash
   kubectl create configmap feature-flags \
       --from-literal=FEATURE_A='true' \
       --from-literal=FEATURE_B='false'
   ```

2. **Verify the ConfigMap**:
   ```bash
   kubectl get configmaps
   ```

### Step 5: Create a Deployment Using ConfigMap

1. **Create a deployment manifest named `deployment.yaml`**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: feature-flags-demo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: feature-flags-demo
     template:
       metadata:
         labels:
           app: feature-flags-demo
       spec:
         containers:
         - name: app
           image: k8s-feature-flags-demo
           ports:
           - containerPort: 5000
           env:
           - name: FEATURE_A
             valueFrom:
               configMapKeyRef:
                 name: feature-flags
                 key: FEATURE_A
           - name: FEATURE_B
             valueFrom:
               configMapKeyRef:
                 name: feature-flags
                 key: FEATURE_B
   ```

   - **Explanation**: This manifest defines a deployment that uses our Docker image, sets environment variables for feature flags from the ConfigMap, and exposes port 5000.

2. **Apply the deployment**:
   ```bash
   kubectl apply -f deployment.yaml
   ```

### Step 6: Expose the Application

1. **Create a service to expose your application**. Create a file named `service.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: feature-flags-demo-service
   spec:
     type: NodePort
     selector:
       app: feature-flags-demo
     ports:
       - port: 5000
         targetPort: 5000
         nodePort: 30001
   ```

2. **Apply the service configuration**:
   ```bash
   kubectl apply -f service.yaml
   ```

### Step 7: Access the Application

1. **Get your Node's IP address**:
   ```bash
   kubectl get nodes -o wide
   ```

2. **Open your browser and navigate to**:
   ```
   http://<node-ip>:30001
   ```
   - Replace `<node-ip>` with the actual IP address you found. You should see:
   ```
   Features:
   Feature A is enabled
   Feature B is disabled
   ```

### Step 8: Update Feature Flags

1. **Update the ConfigMap to change feature flags**:
   ```bash
   kubectl create configmap feature-flags \
       --from-literal=FEATURE_A='false' \
       --from-literal=FEATURE_B='true' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

### Step 9: Verify the Updates

1. **Access the application again**:
   Refresh the browser to see the changes:
   ```
   http://<node-ip>:30001
   ```
   - You should now see:
   ```
   Features:
   Feature A is disabled
   Feature B is enabled
   ```

### Step 10: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service feature-flags-demo-service
kubectl delete deployment feature-flags-demo
kubectl delete configmap feature-flags
```

### Conclusion

In this implementation, we:
1. Created a Flask application that retrieves feature flags from a ConfigMap.
2. Built a Docker image for the application and deployed it in Kubernetes.
3. Exposed the application via a service and accessed it through a web browser.
4. Updated the feature flags in the ConfigMap and verified the changes in the application.

This scenario demonstrates how to manage feature flags dynamically in a Kubernetes environment. If you have any further questions or need additional examples, feel free to ask!
