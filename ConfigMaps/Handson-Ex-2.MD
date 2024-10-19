Sure! Let’s dive deeper into the end-to-end implementation of using a ConfigMap for feature toggles in a real-time example. This will include detailed explanations of each step, allowing you to understand the entire process thoroughly.

### Use Case: Feature Toggles for a Web Application

In this example, we’ll create a simple Node.js web application that features a toggle for a new functionality, managed through a Kubernetes ConfigMap.

### Prerequisites
- A Kubernetes cluster (you can use Minikube, kind, or a cloud provider).
- `kubectl` installed and configured to interact with your cluster.
- Docker installed for building the application image.

### Step 1: Set Up Your Project Directory

1. **Create a directory for your project:**
   ```bash
   mkdir k8s-feature-toggle-demo
   cd k8s-feature-toggle-demo
   ```

### Step 2: Create the ConfigMap for Feature Toggles

ConfigMaps allow you to store configuration data that your applications can access at runtime.

1. **Create a ConfigMap manifest:**
   Create a file named `feature-toggle-configmap.yaml` with the following content:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: feature-toggle
   data:
     ENABLE_NEW_FEATURE: "true"  # This flag controls the feature
   ```

2. **Apply the ConfigMap:**
   ```bash
   kubectl apply -f feature-toggle-configmap.yaml
   ```
   - This command creates the ConfigMap named `feature-toggle` in your Kubernetes cluster.

### Step 3: Create the Sample Application

Now, we’ll create a simple Node.js application that reads the feature toggle from the ConfigMap.

1. **Create the application file:**
   Create a file named `app.js` with the following content:
   ```javascript
   const express = require('express');
   const app = express();
   const port = process.env.PORT || 3000;

   app.get('/', (req, res) => {
       const enableNewFeature = process.env.ENABLE_NEW_FEATURE === 'true';
       let responseMessage = "Welcome to the Feature Toggle Demo!";

       if (enableNewFeature) {
           responseMessage += "\nThe new feature is ENABLED!";
       } else {
           responseMessage += "\nThe new feature is DISABLED.";
       }

       res.send(responseMessage);
   });

   app.listen(port, () => {
       console.log(`App listening at http://localhost:${port}`);
   });
   ```

   - This application serves a simple message that indicates whether a new feature is enabled based on the value of the `ENABLE_NEW_FEATURE` environment variable.

2. **Create a `Dockerfile`:**
   Create a file named `Dockerfile` with the following content:
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
   Create a file named `package.json` with the following content:
   ```json
   {
     "name": "k8s-feature-toggle-demo",
     "version": "1.0.0",
     "main": "app.js",
     "dependencies": {
       "express": "^4.17.1"
     }
   }
   ```

   - This file specifies the application’s dependencies.

### Step 4: Build the Docker Image

1. **Build the Docker image for the application:**
   ```bash
   docker build -t k8s-feature-toggle-demo .
   ```

   - This command builds the Docker image and tags it as `k8s-feature-toggle-demo`.

### Step 5: Create a Deployment Using the ConfigMap

1. **Create a deployment manifest file named `deployment.yaml`:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: feature-toggle-deployment
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: feature-toggle-demo
     template:
       metadata:
         labels:
           app: feature-toggle-demo
       spec:
         containers:
         - name: app
           image: k8s-feature-toggle-demo
           ports:
           - containerPort: 3000
           env:
             - name: ENABLE_NEW_FEATURE
               valueFrom:
                 configMapKeyRef:
                   name: feature-toggle
                   key: ENABLE_NEW_FEATURE
   ```

   - **Explanation**:
     - The deployment is named `feature-toggle-deployment`.
     - It specifies that one replica of the application should run.
     - The application reads the `ENABLE_NEW_FEATURE` environment variable from the `feature-toggle` ConfigMap.

2. **Apply the deployment:**
   ```bash
   kubectl apply -f deployment.yaml
   ```

   - This command creates the deployment in your Kubernetes cluster.

### Step 6: Expose the Application

1. **Create a service to expose your application.** Create a file named `service.yaml` with the following content:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: feature-toggle-service
   spec:
     type: NodePort
     selector:
       app: feature-toggle-demo
     ports:
       - port: 3000
         targetPort: 3000
         nodePort: 30001
   ```

   - **Explanation**:
     - The service is named `feature-toggle-service`.
     - It exposes the application on port `30001` (NodePort) and forwards traffic to port `3000` of the container.

2. **Apply the service configuration:**
   ```bash
   kubectl apply -f service.yaml
   ```

### Step 7: Access the Application

1. **Find your Node's IP address:**
   ```bash
   kubectl get nodes -o wide
   ```

2. **Open your browser and navigate to:**
   ```
   http://<node-ip>:30001
   ```
   - Replace `<node-ip>` with the actual IP address you found. You should see a message indicating whether the new feature is enabled or disabled.

### Step 8: Update the ConfigMap to Toggle the Feature

1. **To disable the feature, update the ConfigMap:**
   ```bash
   kubectl create configmap feature-toggle --from-literal=ENABLE_NEW_FEATURE=false -o yaml --dry-run=client | kubectl apply -f -
   ```

   - This command updates the `ENABLE_NEW_FEATURE` key in the `feature-toggle` ConfigMap to `false`.

2. **Roll out a restart of the deployment to pick up the new ConfigMap values:**
   ```bash
   kubectl rollout restart deployment/feature-toggle-deployment
   ```

3. **Check the application again by refreshing the browser.** You should now see that the new feature is disabled.

### Step 9: Clean Up

Once you're done testing, clean up the resources you created:

```bash
kubectl delete service feature-toggle-service
kubectl delete deployment feature-toggle-deployment
kubectl delete configmap feature-toggle
```

### Conclusion

In this implementation, you:

- Created a ConfigMap to manage feature toggles for a web application.
- Built a simple Node.js application that reads the feature toggle from the ConfigMap.
- Deployed the application in Kubernetes and exposed it via a service.
- Demonstrated how to update the ConfigMap dynamically to toggle features without redeploying the application.

This approach provides a flexible way to manage application behavior, making it easier to adapt features based on user feedback or testing conditions. If you have any questions or need further assistance, feel free to ask!
