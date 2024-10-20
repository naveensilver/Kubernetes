Let's implement a complete end-to-end solution using Kubernetes Ingress for a microservices architecture from scratch, simulating a production-like environment. This example will cover all necessary steps, including deployment, configuration, and testing.

### Use Case: E-Commerce Platform Microservices

We'll deploy three microservices: 
1. **User Authentication Service**
2. **Product Catalog Service**
3. **Payment Processing Service**

We'll also configure an Ingress to route traffic to these services.

### Step-by-Step Implementation

#### Step 1: Set Up Kubernetes Cluster

1. **Install Minikube (if you haven't already):**

   ```bash
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   sudo install minikube-linux-amd64 /usr/local/bin/minikube
   ```

2. **Start Minikube:**

   ```bash
   minikube start
   ```

3. **Enable the Ingress Add-on:**

   ```bash
   minikube addons enable ingress
   ```

#### Step 2: Create Namespace

Create a dedicated namespace for our e-commerce application:

```bash
kubectl create namespace ecommerce
```

#### Step 3: Deploy Microservices

##### 1. User Authentication Service

Create a file named `auth-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
      - name: auth
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=User Authentication Service"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: ecommerce
spec:
  type: ClusterIP
  ports:
  - port: 5678
    targetPort: 5678
  selector:
    app: auth
```

##### 2. Product Catalog Service

Create a file named `catalog-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog
  template:
    metadata:
      labels:
        app: catalog
    spec:
      containers:
      - name: catalog
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=Product Catalog Service"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
  namespace: ecommerce
spec:
  type: ClusterIP
  ports:
  - port: 5678
    targetPort: 5678
  selector:
    app: catalog
```

##### 3. Payment Processing Service

Create a file named `payment-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=Payment Processing Service"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: ecommerce
spec:
  type: ClusterIP
  ports:
  - port: 5678
    targetPort: 5678
  selector:
    app: payment
```

#### Step 4: Deploy the Services

Run the following commands to create the services and deployments in the `ecommerce` namespace:

```bash
kubectl apply -f auth-deployment.yaml
kubectl apply -f catalog-deployment.yaml
kubectl apply -f payment-deployment.yaml
```

#### Step 5: Create Ingress Resource

Create a file named `ingress.yaml` to define the Ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: ecommerce.local
    http:
      paths:
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 5678
      - path: /catalog
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 5678
      - path: /payment
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 5678
```

#### Step 6: Apply the Ingress Resource

Run the following command to create the Ingress resource:

```bash
kubectl apply -f ingress.yaml
```

#### Step 7: Update Local Hosts File

To access the services using the defined hostname, update your local `/etc/hosts` file (or `C:\Windows\System32\drivers\etc\hosts` on Windows) to point `ecommerce.local` to your Minikube IP. Get the Minikube IP:

```bash
minikube ip
```

Add the following line to your hosts file (replace `<MINIKUBE-IP>` with the actual IP address):

```
<MINIKUBE-IP> ecommerce.local
```

#### Step 8: Access the Services

You can now access the services using the following URLs:

- **User Authentication Service**: [http://ecommerce.local/auth](http://ecommerce.local/auth)
- **Product Catalog Service**: [http://ecommerce.local/catalog](http://ecommerce.local/catalog)
- **Payment Processing Service**: [http://ecommerce.local/payment](http://ecommerce.local/payment)

You should see the respective messages for each service when you access these URLs.

#### Step 9: Monitoring the Deployment

You can check the status of your deployments and services using:

```bash
kubectl get deployments -n ecommerce
kubectl get services -n ecommerce
kubectl get ingress -n ecommerce
```

#### Step 10: Cleanup Resources

To delete the Ingress and applications, run:

```bash
kubectl delete -f ingress.yaml
kubectl delete -f auth-deployment.yaml
kubectl delete -f catalog-deployment.yaml
kubectl delete -f payment-deployment.yaml
```

### Conclusion

In this implementation:
- We set up a Kubernetes cluster with an Ingress controller.
- We deployed three microservices for an e-commerce platform.
- An Ingress resource was created to route traffic based on URL paths.
- This setup provides a clean and efficient way to manage external access to multiple services, demonstrating the benefits of using Ingress in a microservices architecture.

If you have further questions or need additional features implemented, feel free to ask!
