To deploy an application end-to-end on Kubernetes from scratch, we will go through a detailed, real-time step-by-step process. We will cover all aspects, starting from setting up the Kubernetes cluster to deploying an application and securing it with an Ingress. I'll explain each step along the way.

### **Prerequisites**
- A working **Kubernetes cluster** (this could be hosted on AWS EKS, GKE, or even a local `minikube` cluster).
- **kubectl** installed and configured to manage your cluster.
- **Docker** installed for containerizing the application.
- An application to deploy (we’ll use a sample **Node.js** application for this example).

---

### **Step 1: Setting Up Namespaces**

We will use different namespaces for different environments (Dev, QA, Prod) to isolate resources.

1. **Create a Namespace for Development**
   ```bash
   kubectl create namespace dev
   ```
2. **Create a Namespace for QA**
   ```bash
   kubectl create namespace qa
   ```
3. **Create a Namespace for Production**
   ```bash
   kubectl create namespace prod
   ```

Namespaces help in organizing resources and managing access controls separately for each environment.

---

### **Step 2: Building & Containerizing the Application**

Assuming we have a simple **Node.js** application, let’s containerize it using Docker. Here’s a sample Dockerfile:

```Dockerfile
# Use Node.js image as the base
FROM node:14

# Set the working directory inside the container
WORKDIR /usr/src/app

# Copy package.json and install dependencies
COPY package*.json ./
RUN npm install

# Copy the application code
COPY . .

# Expose the port on which the app will run
EXPOSE 8080

# Command to start the app
CMD [ "npm", "start" ]
```

1. **Build the Docker image**
   ```bash
   docker build -t myapp:latest .
   ```

2. **Push the Docker image to DockerHub** (or any container registry)
   ```bash
   docker tag myapp:latest <your_dockerhub_username>/myapp:latest
   docker push <your_dockerhub_username>/myapp:latest
   ```

---

### **Step 3: Create ConfigMaps & Secrets**

We will create ConfigMaps to store non-sensitive configurations and Secrets to store sensitive data like credentials.

1. **ConfigMap for application configuration**
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: app-config
     namespace: dev
   data:
     APP_ENV: "development"
     DATABASE_URL: "jdbc:mysql://db:3306/mydb"
   ```

   Apply this ConfigMap:
   ```bash
   kubectl apply -f app-config.yaml
   ```

2. **Secret for sensitive data**
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: db-secret
     namespace: dev
   type: Opaque
   data:
     db_password: cGFzc3dvcmQ= # Base64 encoded password
   ```

   Apply this Secret:
   ```bash
   kubectl apply -f db-secret.yaml
   ```

---

### **Step 4: Persistent Volume & Persistent Volume Claim (for Stateful Applications)**

For stateful applications that require persistent storage, we need to create a PersistentVolume and PersistentVolumeClaim.

1. **PersistentVolume definition**:
   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: myapp-pv
     namespace: dev
   spec:
     capacity:
       storage: 5Gi
     accessModes:
       - ReadWriteOnce
     hostPath:
       path: /mnt/data
   ```

2. **PersistentVolumeClaim definition**:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: myapp-pvc
     namespace: dev
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 5Gi
   ```

Apply both:
```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
```

---

### **Step 5: Deploy the Application using a Deployment**

Now we’ll deploy the application using a Kubernetes **Deployment**. We will configure the deployment to reference the ConfigMap and Secret we created earlier.

1. **Deployment YAML**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp-deployment
     namespace: dev
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: myapp
     template:
       metadata:
         labels:
           app: myapp
       spec:
         containers:
         - name: myapp-container
           image: <your_dockerhub_username>/myapp:latest
           ports:
           - containerPort: 8080
           envFrom:
           - configMapRef:
               name: app-config
           - secretRef:
               name: db-secret
         volumeMounts:
         - name: storage
           mountPath: /usr/src/app/data
         volumes:
         - name: storage
           persistentVolumeClaim:
             claimName: myapp-pvc
   ```

2. **Apply the deployment**:
   ```bash
   kubectl apply -f deployment.yaml
   ```

---

### **Step 6: Expose the Application using a Service**

To expose the application internally (or externally depending on the environment), create a **Service**. This service will route traffic to the application Pods.

1. **ClusterIP Service (internal-only)**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: myapp-service
     namespace: dev
   spec:
     selector:
       app: myapp
     ports:
     - protocol: TCP
       port: 80
       targetPort: 8080
   ```

2. **NodePort Service (if external access is needed)**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: myapp-service
     namespace: dev
   spec:
     type: NodePort
     selector:
       app: myapp
     ports:
     - protocol: TCP
       port: 80
       targetPort: 8080
       nodePort: 30007
   ```

3. **Apply the service**:
   ```bash
   kubectl apply -f service.yaml
   ```

---

### **Step 7: Configuring Ingress (Optional)**

To expose your service externally via an HTTP URL, you can use **Ingress** with a domain name (e.g., `myapp.dev.com`).

1. **Ingress YAML**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: myapp-ingress
     namespace: dev
   spec:
     rules:
     - host: myapp.dev.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: myapp-service
               port:
                 number: 80
   ```

2. **Apply the ingress**:
   ```bash
   kubectl apply -f ingress.yaml
   ```

3. **Configure DNS**: Point your domain (e.g., `myapp.dev.com`) to the external IP of your Kubernetes cluster or load balancer.

---

### **Step 8: Monitoring and Scaling (Optional)**

1. **Set up Horizontal Pod Autoscaler (HPA)** for automatic scaling:
   ```bash
   kubectl autoscale deployment myapp-deployment --cpu-percent=50 --min=1 --max=10 -n dev
   ```

2. **Monitoring**: Integrate Prometheus and Grafana to monitor your application health and performance metrics.

---

### **End-to-End Application Deployment Summary**
1. Set up **Namespaces** for environment isolation.
2. **Containerize** the application using Docker and push to a container registry.
3. Create **ConfigMaps** and **Secrets** to manage configuration.
4. Set up **Persistent Storage** using PVs and PVCs for stateful apps.
5. Deploy the application using **Deployments** and reference ConfigMaps, Secrets, and PVCs.
6. Expose the application using **Services** (ClusterIP, NodePort, or LoadBalancer).
7. Optionally configure **Ingress** to expose the application via an HTTP URL.
8. Set up **monitoring** and **scaling** mechanisms for production readiness.

This process is the foundation of deploying real-time applications on Kubernetes. You can modify and adapt it based on the specific requirements of your project.
