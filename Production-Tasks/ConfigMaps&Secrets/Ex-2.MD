Let’s go through a comprehensive, step-by-step implementation of using ConfigMaps and Secrets in Kubernetes, focusing on a practical scenario with a Python Flask application. This will include detailed explanations and hands-on steps to ensure clarity.

### Use Case: Configuration for a Flask Application

We'll create a Flask application that:
- Retrieves configuration settings (like API URL) from a ConfigMap.
- Retrieves sensitive data (like API keys) from a Secret.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-flask-api-demo
   cd k8s-flask-api-demo
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
       api_url = os.environ.get('API_URL', 'http://default.api')
       api_key = os.environ.get('API_KEY', 'default_key')
       return f"API URL: {api_url}, API Key: {api_key}"

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

   - **Explanation**: This code creates a simple Flask web application that reads the `API_URL` and `API_KEY` from environment variables and displays them when accessed.

2. **Create a `requirements.txt` file**:
   ```
   Flask==2.0.1
   ```

   - **Explanation**: This file specifies the dependencies for our Flask application.

3. **Create a Dockerfile**:
   ```dockerfile
   FROM python:3.9

   WORKDIR /usr/src/app
   COPY requirements.txt ./
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .

   CMD ["python", "app.py"]
   ```

   - **Explanation**: This Dockerfile sets up a Python environment, installs dependencies, and runs the Flask application.

### Step 3: Build the Docker Image

1. **Build the Docker image**:
   ```bash
   docker build -t k8s-flask-api-demo .
   ```

   - **Explanation**: This command builds the Docker image and tags it as `k8s-flask-api-demo`.

### Step 4: Create a ConfigMap

1. **Create a ConfigMap for the API configuration**:
   ```bash
   kubectl create configmap api-config \
       --from-literal=API_URL='https://api.example.com/v1'
   ```

   - **Explanation**: This command creates a ConfigMap named `api-config` containing the API URL.

2. **Verify the ConfigMap**:
   ```bash
   kubectl get configmaps
   ```

### Step 5: Create a Secret

1. **Create a Secret for the API key**:
   ```bash
   kubectl create secret generic api-key-secret \
       --from-literal=api_key='my-secret-api-key'
   ```

   - **Explanation**: This command creates a Secret named `api-key-secret` that contains the sensitive API key.

2. **Verify the Secret**:
   ```bash
   kubectl get secrets
   ```

### Step 6: Create a Deployment Using ConfigMap and Secret

1. **Create a deployment manifest named `deployment.yaml`**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-api-demo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: flask-api-demo
     template:
       metadata:
         labels:
           app: flask-api-demo
       spec:
         containers:
         - name: app
           image: k8s-flask-api-demo
           ports:
           - containerPort: 5000
           env:
           - name: API_URL
             valueFrom:
               configMapKeyRef:
                 name: api-config
                 key: API_URL
           - name: API_KEY
             valueFrom:
               secretKeyRef:
                 name: api-key-secret
                 key: api_key
   ```

   - **Explanation**: This manifest defines a deployment that uses our Docker image, sets environment variables from the ConfigMap and Secret, and exposes port 5000.

2. **Apply the deployment**:
   ```bash
   kubectl apply -f deployment.yaml
   ```

### Step 7: Expose the Application

1. **Create a service to expose your application**. Create a file named `service.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-api-demo-service
   spec:
     type: NodePort
     selector:
       app: flask-api-demo
     ports:
       - port: 5000
         targetPort: 5000
         nodePort: 30001
   ```

   - **Explanation**: This service will expose our Flask application on port 30001.

2. **Apply the service configuration**:
   ```bash
   kubectl apply -f service.yaml
   ```

### Step 8: Access the Application

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
   API URL: https://api.example.com/v1, API Key: my-secret-api-key
   ```

### Step 9: Update ConfigMap and Secret

1. **Update the ConfigMap**:
   ```bash
   kubectl create configmap api-config \
       --from-literal=API_URL='https://api.updatedexample.com/v2' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

   - **Explanation**: This command updates the ConfigMap with a new API URL.

2. **Update the Secret**:
   ```bash
   kubectl create secret generic api-key-secret \
       --from-literal=api_key='new-secret-api-key' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

   - **Explanation**: This command updates the Secret with a new API key.

### Step 10: Verify the Updates

1. **Access the application again**:
   Refresh the browser to see the changes:
   ```
   http://<node-ip>:30001
   ```
   - You should now see:
   ```
   API URL: https://api.updatedexample.com/v2, API Key: new-secret-api-key
   ```

### Step 11: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service flask-api-demo-service
kubectl delete deployment flask-api-demo
kubectl delete configmap api-config
kubectl delete secret api-key-secret
```

### Conclusion

In this implementation, we:
1. Created a Flask application that retrieves configuration settings and sensitive data from a ConfigMap and a Secret.
2. Built a Docker image for the application and deployed it in Kubernetes.
3. Exposed the application via a service and accessed it through a web browser.
4. Updated the ConfigMap and Secret, demonstrating how to manage configuration and secrets dynamically.

This scenario illustrates effective management of application configuration and sensitive information in a Kubernetes environment. If you have any further questions or need additional details, feel free to ask!
