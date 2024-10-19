Let’s go through a detailed end-to-end implementation of the third use case: **External API Configuration** using ConfigMaps and Secrets in Kubernetes. This will allow us to manage API endpoint configurations and credentials securely.

### Use Case: External API Configuration

In this scenario:
- We'll use a **ConfigMap** to store the external API base URL.
- We'll use a **Secret** to store the API key needed for authentication.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-external-api-demo
   cd k8s-external-api-demo
   ```

### Step 2: Create the Python Application

1. **Create the application code**:
   - **Create a file named `app.py`**:
   ```python
   from flask import Flask, jsonify
   import os
   import requests

   app = Flask(__name__)

   @app.route('/api/data')
   def get_data():
       api_url = os.environ.get('API_URL', 'https://api.example.com/data')
       api_key = os.environ.get('API_KEY', 'default_key')
       
       headers = {'Authorization': f'Bearer {api_key}'}
       response = requests.get(api_url, headers=headers)
       
       if response.status_code == 200:
           return jsonify(response.json())
       else:
           return jsonify({"error": "Unable to fetch data"}), response.status_code

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

   - **Explanation**: This application defines an endpoint `/api/data` that fetches data from an external API. It retrieves the API URL and key from environment variables and uses them to make the request.

2. **Create a `requirements.txt` file**:
   ```
   Flask==2.0.1
   requests==2.26.0
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
   docker build -t k8s-external-api-demo .
   ```

### Step 4: Create a ConfigMap for API Configuration

1. **Create a ConfigMap for the external API configuration**:
   ```bash
   kubectl create configmap api-config \
       --from-literal=API_URL='https://api.example.com/data'
   ```

2. **Verify the ConfigMap**:
   ```bash
   kubectl get configmaps
   ```

### Step 5: Create a Secret for API Credentials

1. **Create a Secret for the API key**:
   ```bash
   kubectl create secret generic api-key-secret \
       --from-literal=api_key='my-secret-api-key'
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
     name: external-api-demo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: external-api-demo
     template:
       metadata:
         labels:
           app: external-api-demo
       spec:
         containers:
         - name: app
           image: k8s-external-api-demo
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

   - **Explanation**: This manifest defines a deployment that uses our Docker image, sets environment variables for the API URL from the ConfigMap and the API key from the Secret, and exposes port 5000.

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
     name: external-api-demo-service
   spec:
     type: NodePort
     selector:
       app: external-api-demo
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
   http://<node-ip>:30001/api/data
   ```
   - Replace `<node-ip>` with the actual IP address you found. You should see either a successful JSON response from the external API or an error message.

### Step 9: Update API Configuration

1. **Update the ConfigMap**:
   ```bash
   kubectl create configmap api-config \
       --from-literal=API_URL='https://api.newexample.com/data' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

2. **Update the Secret (if needed)**:
   ```bash
   kubectl create secret generic api-key-secret \
       --from-literal=api_key='my-new-secret-api-key' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

### Step 10: Verify the Updates

1. **Access the application again**:
   Refresh the API endpoint in your browser:
   ```
   http://<node-ip>:30001/api/data
   ```
   - You should now see a response from the new API URL if it is accessible.

### Step 11: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service external-api-demo-service
kubectl delete deployment external-api-demo
kubectl delete configmap api-config
kubectl delete secret api-key-secret
```

### Conclusion

In this implementation, we:
1. Created a Flask application that retrieves API configuration settings from a ConfigMap and sensitive API credentials from a Secret.
2. Built a Docker image for the application and deployed it in Kubernetes.
3. Exposed the application via a service and accessed the API data through a web browser.
4. Updated the ConfigMap and Secret to demonstrate dynamic management of configurations.

This scenario illustrates effective management of external API configurations and sensitive information in a Kubernetes environment. If you have any further questions or need additional examples, feel free to ask!
