Let’s dive into the fourth use case: **Application Settings** using ConfigMaps and Secrets in Kubernetes. This use case will demonstrate how to manage various application-specific configurations such as logging levels, service URLs, and other parameters.

### Use Case: Application Settings

In this scenario:
- We'll use a **ConfigMap** to store various application settings, including a logging level and an external service URL.
- We'll use a **Secret** to store a sensitive API token for authenticating with an external service.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-app-settings-demo
   cd k8s-app-settings-demo
   ```

### Step 2: Create the Python Application

1. **Create the application code**:
   - **Create a file named `app.py`**:
   ```python
   from flask import Flask, jsonify
   import os

   app = Flask(__name__)

   @app.route('/config')
   def config():
       logging_level = os.environ.get('LOGGING_LEVEL', 'INFO')
       external_service_url = os.environ.get('EXTERNAL_SERVICE_URL', 'https://api.example.com')
       api_token = os.environ.get('API_TOKEN', 'default_token')

       return jsonify({
           "logging_level": logging_level,
           "external_service_url": external_service_url,
           "api_token": api_token
       })

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

   - **Explanation**: This application defines an endpoint `/config` that returns the application's logging level, external service URL, and API token.

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
   docker build -t k8s-app-settings-demo .
   ```

### Step 4: Create a ConfigMap for Application Settings

1. **Create a ConfigMap for the application settings**:
   ```bash
   kubectl create configmap app-config \
       --from-literal=LOGGING_LEVEL='DEBUG' \
       --from-literal=EXTERNAL_SERVICE_URL='https://api.example.com'
   ```

2. **Verify the ConfigMap**:
   ```bash
   kubectl get configmaps
   ```

### Step 5: Create a Secret for Sensitive API Token

1. **Create a Secret for the API token**:
   ```bash
   kubectl create secret generic api-token-secret \
       --from-literal=api_token='my-secret-api-token'
   ```

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
     name: app-settings-demo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: app-settings-demo
     template:
       metadata:
         labels:
           app: app-settings-demo
       spec:
         containers:
         - name: app
           image: k8s-app-settings-demo
           ports:
           - containerPort: 5000
           env:
           - name: LOGGING_LEVEL
             valueFrom:
               configMapKeyRef:
                 name: app-config
                 key: LOGGING_LEVEL
           - name: EXTERNAL_SERVICE_URL
             valueFrom:
               configMapKeyRef:
                 name: app-config
                 key: EXTERNAL_SERVICE_URL
           - name: API_TOKEN
             valueFrom:
               secretKeyRef:
                 name: api-token-secret
                 key: api_token
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
     name: app-settings-demo-service
   spec:
     type: NodePort
     selector:
       app: app-settings-demo
     ports:
       - port: 5000
         targetPort: 5000
         nodePort: 30001
   ```

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
   http://<node-ip>:30001/config
   ```
   - Replace `<node-ip>` with the actual IP address you found. You should see a JSON response with the application settings:
   ```json
   {
       "logging_level": "DEBUG",
       "external_service_url": "https://api.example.com",
       "api_token": "my-secret-api-token"
   }
   ```

### Step 9: Update Application Settings

1. **Update the ConfigMap** to change the logging level and external service URL:
   ```bash
   kubectl create configmap app-config \
       --from-literal=LOGGING_LEVEL='INFO' \
       --from-literal=EXTERNAL_SERVICE_URL='https://api.newexample.com' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

2. **Update the Secret (if needed)**:
   ```bash
   kubectl create secret generic api-token-secret \
       --from-literal=api_token='my-new-secret-api-token' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

### Step 10: Verify the Updates

1. **Access the application again**:
   Refresh the configuration endpoint in your browser:
   ```
   http://<node-ip>:30001/config
   ```
   - You should now see the updated settings:
   ```json
   {
       "logging_level": "INFO",
       "external_service_url": "https://api.newexample.com",
       "api_token": "my-new-secret-api-token"
   }
   ```

### Step 11: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service app-settings-demo-service
kubectl delete deployment app-settings-demo
kubectl delete configmap app-config
kubectl delete secret api-token-secret
```

### Conclusion

In this implementation, we:
1. Created a Flask application that retrieves application settings from a ConfigMap and a sensitive API token from a Secret.
2. Built a Docker image for the application and deployed it in Kubernetes.
3. Exposed the application via a service and accessed the configuration through a web browser.
4. Updated the ConfigMap and Secret to demonstrate dynamic management of application settings.

This scenario illustrates effective management of application configurations and sensitive information in a Kubernetes environment. If you have any further questions or need additional examples, feel free to ask!
