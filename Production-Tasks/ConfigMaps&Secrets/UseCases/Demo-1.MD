Let’s go through a detailed end-to-end implementation of the first use case: **Database Configuration** using ConfigMaps and Secrets in Kubernetes. We'll create a simple Python Flask application that connects to a PostgreSQL database. 

### Use Case: Database Configuration

In this scenario:
- We'll use a **ConfigMap** to store database connection parameters (like the host, port, and database name).
- We'll use a **Secret** to store the database username and password.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-db-config-demo
   cd k8s-db-config-demo
   ```

### Step 2: Create the Python Application

1. **Create the application code**:
   - **Create a file named `app.py`**:
   ```python
   from flask import Flask
   import os
   import psycopg2

   app = Flask(__name__)

   @app.route('/')
   def index():
       try:
           # Get database configurations from environment variables
           db_host = os.environ.get('DB_HOST', 'localhost')
           db_port = os.environ.get('DB_PORT', '5432')
           db_name = os.environ.get('DB_NAME', 'testdb')
           db_user = os.environ.get('DB_USER', 'user')
           db_password = os.environ.get('DB_PASSWORD', 'password')

           # Connect to the PostgreSQL database
           connection = psycopg2.connect(
               host=db_host,
               port=db_port,
               database=db_name,
               user=db_user,
               password=db_password
           )
           return f"Connected to the database {db_name} at {db_host}:{db_port} as {db_user}"
       except Exception as e:
           return str(e)

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

   - **Explanation**: This application connects to a PostgreSQL database and retrieves database connection details from environment variables.

2. **Create a `requirements.txt` file**:
   ```
   Flask==2.0.1
   psycopg2-binary==2.9.1
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
   docker build -t k8s-db-config-demo .
   ```

### Step 4: Create a ConfigMap

1. **Create a ConfigMap for the database configuration**:
   ```bash
   kubectl create configmap db-config \
       --from-literal=DB_HOST='db-host' \
       --from-literal=DB_PORT='5432' \
       --from-literal=DB_NAME='testdb'
   ```

2. **Verify the ConfigMap**:
   ```bash
   kubectl get configmaps
   ```

### Step 5: Create a Secret

1. **Create a Secret for the database credentials**:
   ```bash
   kubectl create secret generic db-credentials \
       --from-literal=username='myuser' \
       --from-literal=password='mypassword'
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
     name: db-config-demo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: db-config-demo
     template:
       metadata:
         labels:
           app: db-config-demo
       spec:
         containers:
         - name: app
           image: k8s-db-config-demo
           ports:
           - containerPort: 5000
           env:
           - name: DB_HOST
             valueFrom:
               configMapKeyRef:
                 name: db-config
                 key: DB_HOST
           - name: DB_PORT
             valueFrom:
               configMapKeyRef:
                 name: db-config
                 key: DB_PORT
           - name: DB_NAME
             valueFrom:
               configMapKeyRef:
                 name: db-config
                 key: DB_NAME
           - name: DB_USER
             valueFrom:
               secretKeyRef:
                 name: db-credentials
                 key: username
           - name: DB_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: db-credentials
                 key: password
   ```

   - **Explanation**: This manifest defines a deployment that uses the Docker image we built, sets environment variables from the ConfigMap and Secret, and exposes port 5000.

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
     name: db-config-demo-service
   spec:
     type: NodePort
     selector:
       app: db-config-demo
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
   http://<node-ip>:30001
   ```
   - Replace `<node-ip>` with the actual IP address you found. You should see:
   ```
   Connected to the database testdb at db-host:5432 as myuser
   ```

### Step 9: Update ConfigMap and Secret

1. **Update the ConfigMap**:
   ```bash
   kubectl create configmap db-config \
       --from-literal=DB_HOST='new-db-host' \
       --from-literal=DB_PORT='5432' \
       --from-literal=DB_NAME='newtestdb' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

2. **Update the Secret**:
   ```bash
   kubectl create secret generic db-credentials \
       --from-literal=username='newuser' \
       --from-literal=password='newpassword' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

### Step 10: Verify the Updates

1. **Access the application again**:
   Refresh the browser to see the changes:
   ```
   http://<node-ip>:30001
   ```
   - You should now see:
   ```
   Connected to the database newtestdb at new-db-host:5432 as newuser
   ```

### Step 11: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service db-config-demo-service
kubectl delete deployment db-config-demo
kubectl delete configmap db-config
kubectl delete secret db-credentials
```

### Conclusion

In this implementation, we:
1. Created a Flask application that retrieves database configuration settings from a ConfigMap and sensitive credentials from a Secret.
2. Built a Docker image for the application and deployed it in Kubernetes.
3. Exposed the application via a service and accessed it through a web browser.
4. Updated the ConfigMap and Secret, demonstrating dynamic management of configurations and credentials.

This scenario highlights effective management of database configurations and sensitive information in a Kubernetes environment. If you have any further questions or need additional examples, feel free to ask!
