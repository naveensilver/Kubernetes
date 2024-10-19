Let’s implement the use case for **Managing TLS Certificates** using Kubernetes Secrets. This example will demonstrate how to secure a web application with HTTPS by using TLS certificates stored in Secrets.

### Use Case: Managing TLS Certificates

In this scenario:
- We’ll store the TLS certificate and private key in a Kubernetes Secret.
- A Flask web application will be configured to use these certificates for HTTPS communication.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-tls-demo
   cd k8s-tls-demo
   ```

### Step 2: Create TLS Certificates

1. **Generate a self-signed TLS certificate and key**:
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout tls.key -x509 -days 365 -out tls.crt -subj "/CN=localhost"
   ```

   This command will create two files: `tls.crt` (the certificate) and `tls.key` (the private key).

### Step 3: Create a Kubernetes Secret for TLS Certificates

1. **Create a Secret to store the TLS certificate and key**:
   ```bash
   kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
   ```

2. **Verify the Secret**:
   ```bash
   kubectl get secrets
   ```

### Step 4: Create the Python Application

1. **Create the application code**:
   - **Create a file named `app.py`**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def index():
       return jsonify({"message": "Hello, secure world!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000, ssl_context=('/etc/ssl/tls.crt', '/etc/ssl/tls.key'))
   ```

   - **Explanation**: This application serves a simple JSON message over HTTPS. The `ssl_context` parameter specifies the paths to the TLS certificate and private key.

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

### Step 5: Build the Docker Image

1. **Build the Docker image**:
   ```bash
   docker build -t k8s-tls-demo .
   ```

### Step 6: Create a Deployment Using the TLS Secret

1. **Create a deployment manifest named `deployment.yaml`**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: tls-demo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: tls-demo
     template:
       metadata:
         labels:
           app: tls-demo
       spec:
         containers:
         - name: app
           image: k8s-tls-demo
           ports:
           - containerPort: 5000
           volumeMounts:
           - name: tls-volume
             mountPath: /etc/ssl
         volumes:
         - name: tls-volume
           secret:
             secretName: tls-secret
   ```

   - **Explanation**: This manifest defines a deployment that uses our Docker image. It mounts the TLS Secret as a volume at `/etc/ssl`, where the application can access the certificate and key.

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
     name: tls-demo-service
   spec:
     type: NodePort
     selector:
       app: tls-demo
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
   https://<node-ip>:30001/
   ```
   - Replace `<node-ip>` with the actual IP address you found. You may see a warning in your browser because the certificate is self-signed. You can proceed to the site (usually by clicking "Advanced" and "Proceed").

   You should see a JSON response:
   ```json
   {
       "message": "Hello, secure world!"
   }
   ```

### Step 9: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service tls-demo-service
kubectl delete deployment tls-demo
kubectl delete secret tls-secret
```

### Conclusion

In this implementation, we:
1. Generated self-signed TLS certificates and stored them in a Kubernetes Secret.
2. Created a Flask application that serves HTTPS traffic using those certificates.
3. Built a Docker image for the application and deployed it in Kubernetes.
4. Exposed the application via a service and accessed it securely through HTTPS.

This use case demonstrates how to manage TLS certificates and secure communication in a Kubernetes environment. If you have any further questions or need additional examples, feel free to ask!
