Let’s walk through the end-to-end implementation of using ConfigMaps in Kubernetes with detailed explanations at each step. 

### Prerequisites
1. **Kubernetes Cluster**: Ensure you have access to a Kubernetes cluster. You can set up a local cluster using tools like Minikube or kind.
2. **kubectl**: Make sure you have `kubectl` installed and configured to connect to your Kubernetes cluster.

### Step 1: Set Up Your Project Directory

1. **Open your terminal** and create a directory for your project:
   ```bash
   mkdir k8s-configmap-demo
   cd k8s-configmap-demo
   ```

### Step 2: Create a ConfigMap

ConfigMaps store configuration data in key-value pairs.

1. **Create a ConfigMap from literal values:**
   ```bash
   kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=APP_LOG_LEVEL=info
   ```
   - `app-config`: Name of the ConfigMap.
   - `APP_ENV` and `APP_LOG_LEVEL`: Key-value pairs stored in the ConfigMap.

2. **Verify the ConfigMap was created:**
   ```bash
   kubectl get configmaps
   ```
   This command should display `app-config` in the list of ConfigMaps.

3. **Describe the ConfigMap for details:**
   ```bash
   kubectl describe configmap app-config
   ```
   This will show the keys and values stored in the ConfigMap.

### Step 3: Create a Sample Application

We'll create a simple Node.js application that uses the configuration stored in the ConfigMap.

1. **Create a file named `app.js`:**
   ```javascript
   const express = require('express');
   const app = express();
   const port = process.env.PORT || 3000;

   app.get('/', (req, res) => {
       const env = process.env.APP_ENV || 'development';
       const logLevel = process.env.APP_LOG_LEVEL || 'debug';
       res.send(`Environment: ${env}, Log Level: ${logLevel}`);
   });

   app.listen(port, () => {
       console.log(`App listening at http://localhost:${port}`);
   });
   ```

   - This application serves a simple message displaying the environment and log level based on the environment variables.

2. **Create a `Dockerfile`:**
   ```dockerfile
   FROM node:14

   WORKDIR /usr/src/app
   COPY package*.json ./
   RUN npm install
   COPY . .

   CMD ["node", "app.js"]
   ```

   - This Dockerfile specifies how to build the Node.js application image.

3. **Create a `package.json` file:**
   ```json
   {
     "name": "k8s-configmap-demo",
     "version": "1.0.0",
     "main": "app.js",
     "dependencies": {
       "express": "^4.17.1"
     }
   }
   ```

   - This file specifies the dependencies for the application.

### Step 4: Build the Docker Image

1. **Build the Docker image:**
   ```bash
   docker build -t k8s-configmap-demo .
   ```

   - This command builds the Docker image with the tag `k8s-configmap-demo` using the current directory (`.`) as the context.

### Step 5: Create a Deployment to Use the ConfigMap

1. **Create a deployment manifest file named `deployment.yaml`:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: app-deployment
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: configmap-demo
     template:
       metadata:
         labels:
           app: configmap-demo
       spec:
         containers:
         - name: app
           image: k8s-configmap-demo
           ports:
           - containerPort: 3000
           env:
             - name: APP_ENV
               valueFrom:
                 configMapKeyRef:
                   name: app-config
                   key: APP_ENV
             - name: APP_LOG_LEVEL
               valueFrom:
                 configMapKeyRef:
                   name: app-config
                   key: APP_LOG_LEVEL
   ```

   - **Explanation**:
     - The deployment is named `app-deployment`.
     - It specifies one replica (one instance of the app).
     - It uses the `app-config` ConfigMap to set environment variables for `APP_ENV` and `APP_LOG_LEVEL`.

2. **Apply the deployment:**
   ```bash
   kubectl apply -f deployment.yaml
   ```

   - This command creates the deployment in your Kubernetes cluster.

3. **Verify the deployment is running:**
   ```bash
   kubectl get pods
   ```
   You should see a pod named something like `app-deployment-xxxxxxx`.

4. **Check the logs of the pod to see the output:**
   ```bash
   kubectl logs <pod-name>
   ```

### Step 6: Expose the Application

1. **Create a service to expose your application:**
   Create a file named `service.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: app-service
   spec:
     type: NodePort
     selector:
       app: configmap-demo
     ports:
       - port: 3000
         targetPort: 3000
         nodePort: 30001
   ```

   - **Explanation**:
     - The service is named `app-service`.
     - It exposes the application on port `30001` (NodePort).

2. **Apply the service configuration:**
   ```bash
   kubectl apply -f service.yaml
   ```

3. **Get the service details:**
   ```bash
   kubectl get services
   ```
   You should see `app-service` with a `NodePort` listed.

### Step 7: Access the Application

1. **Find your Node's IP address:**
   ```bash
   kubectl get nodes -o wide
   ```

2. **Open your browser and navigate to:**
   ```
   http://<node-ip>:30001
   ```
   - Replace `<node-ip>` with the actual IP address you found.

3. **You should see a message displaying the environment and log level, e.g.,**
   ```
   Environment: production, Log Level: info
   ```

### Step 8: Update the ConfigMap

If you want to change the log level in the ConfigMap, you can do so without changing the deployment.

1. **Update the ConfigMap:**
   ```bash
   kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=APP_LOG_LEVEL=debug -o yaml --dry-run=client | kubectl apply -f -
   ```

2. **Roll out a restart of the deployment to pick up the new ConfigMap values:**
   ```bash
   kubectl rollout restart deployment/app-deployment
   ```

3. **Check the logs again to see the updated log level:**
   ```bash
   kubectl logs -f deployment/app-deployment
   ```

### Step 9: Clean Up

Once you're done, clean up the resources you created:

```bash
kubectl delete service app-service
kubectl delete deployment app-deployment
kubectl delete configmap app-config
```

### Conclusion

You’ve successfully implemented a complete workflow using ConfigMaps in Kubernetes. You learned how to:

1. Create a ConfigMap to store application configuration.
2. Build a simple Node.js application that uses the ConfigMap.
3. Deploy the application in Kubernetes and expose it via a service.
4. Update the ConfigMap and restart the deployment to reflect changes.
