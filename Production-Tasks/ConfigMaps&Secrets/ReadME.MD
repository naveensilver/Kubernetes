Let’s go through an end-to-end implementation of using both ConfigMaps and Secrets in Kubernetes, with detailed explanations and hands-on steps to help you understand the process thoroughly.

### Use Case: Web Application Configuration and Secrets Management

We'll create a Node.js web application that:
- Retrieves application configuration settings from a ConfigMap.
- Retrieves sensitive data (like database credentials) from a Secret.

### Step 1: Set Up Your Project Directory

1. **Create a project directory**:
   ```bash
   mkdir k8s-configmap-secret-demo
   cd k8s-configmap-secret-demo
   ```

### Step 2: Create the Node.js Application

1. **Create the application code**:
   - **Create a file named `app.js`**:
   ```javascript
   const express = require('express');
   const app = express();
   const port = process.env.PORT || 3000;

   const appName = process.env.APP_NAME || "Default App";
   const appVersion = process.env.APP_VERSION || "1.0.0";
   const dbUser = process.env.DB_USER || "defaultUser";
   const dbPassword = process.env.DB_PASSWORD || "defaultPassword";

   app.get('/', (req, res) => {
       res.send(`Welcome to ${appName} (v${appVersion})! <br> Database User: ${dbUser}, Password: ${dbPassword}`);
   });

   app.listen(port, () => {
       console.log(`App listening at http://localhost:${port}`);
   });
   ```

   - This application will read environment variables for the application name, version, and database credentials.

2. **Create a `package.json` file**:
   ```json
   {
     "name": "k8s-configmap-secret-demo",
     "version": "1.0.0",
     "main": "app.js",
     "dependencies": {
       "express": "^4.17.1"
     }
   }
   ```

3. **Create a Dockerfile**:
   ```dockerfile
   FROM node:14

   WORKDIR /usr/src/app
   COPY package*.json ./
   RUN npm install
   COPY . .

   CMD ["node", "app.js"]
   ```

### Step 3: Build the Docker Image

1. **Build the Docker image**:
   ```bash
   docker build -t k8s-configmap-secret-demo .
   ```

   - This command creates a Docker image tagged as `k8s-configmap-secret-demo`.

### Step 4: Create a ConfigMap

1. **Create a ConfigMap for application configuration**:
   ```bash
   kubectl create configmap app-config \
       --from-literal=APP_NAME='My Awesome App' \
       --from-literal=APP_VERSION='1.0.0'
   ```

   - **Explanation**:
     - This command creates a ConfigMap named `app-config` with two key-value pairs: `APP_NAME` and `APP_VERSION`.

2. **Verify the ConfigMap**:
   ```bash
   kubectl get configmaps
   ```

### Step 5: Create a Secret

1. **Create a Secret for database credentials**:
   ```bash
   kubectl create secret generic db-credentials \
       --from-literal=username=my-db-user \
       --from-literal=password=my-db-password
   ```

   - **Explanation**:
     - This command creates a Secret named `db-credentials` containing two sensitive values: `username` and `password`.

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
     name: configmap-secret-demo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: configmap-secret-demo
     template:
       metadata:
         labels:
           app: configmap-secret-demo
       spec:
         containers:
         - name: app
           image: k8s-configmap-secret-demo
           ports:
           - containerPort: 3000
           env:
           - name: APP_NAME
             valueFrom:
               configMapKeyRef:
                 name: app-config
                 key: APP_NAME
           - name: APP_VERSION
             valueFrom:
               configMapKeyRef:
                 name: app-config
                 key: APP_VERSION
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

   - **Explanation**:
     - This manifest defines a deployment named `configmap-secret-demo` with one replica.
     - The deployment uses the Docker image we built earlier.
     - It sets environment variables from the ConfigMap and Secret.

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
     name: configmap-secret-demo-service
   spec:
     type: NodePort
     selector:
       app: configmap-secret-demo
     ports:
       - port: 3000
         targetPort: 3000
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
   Welcome to My Awesome App (v1.0.0)! 
   Database User: my-db-user, Password: my-db-password
   ```

### Step 9: Update ConfigMap and Secret

1. **Update the ConfigMap**:
   ```bash
   kubectl create configmap app-config \
       --from-literal=APP_NAME='My Updated App' \
       --from-literal=APP_VERSION='1.1.0' \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

2. **Update the Secret**:
   ```bash
   kubectl create secret generic db-credentials \
       --from-literal=username=new-db-user \
       --from-literal=password=new-db-password \
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
   Welcome to My Updated App (v1.1.0)! 
   Database User: new-db-user, Password: new-db-password
   ```

### Step 11: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service configmap-secret-demo-service
kubectl delete deployment configmap-secret-demo
kubectl delete configmap app-config
kubectl delete secret db-credentials
```

### Conclusion

In this detailed implementation, we:

1. Created a Node.js application that retrieves configuration from a ConfigMap and sensitive data from a Secret.
2. Built a Docker image and deployed it in Kubernetes.
3. Exposed the application and accessed it via a web browser.
4. Updated the ConfigMap and Secret, demonstrating how to manage configuration and sensitive data dynamically.

This workflow allows for effective management of application settings and sensitive information securely in a Kubernetes environment. 
