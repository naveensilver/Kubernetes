Let’s walk through a detailed, end-to-end implementation of a network policy for a microservices architecture in Kubernetes. In this example, we’ll set up a simple scenario with three services: a frontend service, a backend service, and a database service. We’ll use network policies to control communication between these services.

### Prerequisites

1. **Kubernetes Cluster**: You should have a running Kubernetes cluster (minikube, kind, or a managed Kubernetes service).
2. **kubectl**: Make sure `kubectl` is configured to access your cluster.

### Step 1: Deploy the Microservices

#### 1.1 Create a Namespace

First, create a namespace to isolate our services.

```bash
kubectl create namespace microservices-demo
```

#### 1.2 Deploy the Database Service

Create a deployment and service for the database (using a simple SQLite database for this example).

**database-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: microservices-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: database
        image: nouchka/sqlite3
        ports:
        - containerPort: 8080
        command: ["/bin/sh", "-c", "tail -f /dev/null"]
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: microservices-demo
spec:
  selector:
    app: database
  ports:
  - port: 8080
    targetPort: 8080
```

Apply it:

```bash
kubectl apply -f database-deployment.yaml
```

#### 1.3 Deploy the Backend Service

Create a simple backend service.

**backend-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: microservices-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: httpd:2.4
        ports:
        - containerPort: 80
        command: ["httpd-foreground"]
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: microservices-demo
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

Apply it:

```bash
kubectl apply -f backend-deployment.yaml
```

#### 1.4 Deploy the Frontend Service

Create a simple frontend service.

**frontend-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: microservices-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: httpd:2.4
        ports:
        - containerPort: 80
        command: ["httpd-foreground"]
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: microservices-demo
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

Apply it:

```bash
kubectl apply -f frontend-deployment.yaml
```

### Step 2: Create Network Policies

#### 2.1 Allow Frontend to Backend

Now, let’s create a network policy that allows traffic from the frontend to the backend.

**frontend-to-backend-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: microservices-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

Apply the network policy:

```bash
kubectl apply -f frontend-to-backend-policy.yaml
```

#### 2.2 Deny Frontend to Database

Next, we’ll ensure the frontend cannot access the database directly.

**deny-frontend-to-database-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-frontend-to-database
  namespace: microservices-demo
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    # No rules specified means deny by default
```

Apply the network policy:

```bash
kubectl apply -f deny-frontend-to-database-policy.yaml
```

### Step 3: Testing the Network Policies

To test our setup, we can use `kubectl exec` to run commands in the pods.

#### 3.1 Check Frontend to Backend Communication

First, get a shell in the frontend pod:

```bash
kubectl exec -it $(kubectl get pods -n microservices-demo -l app=frontend -o jsonpath='{.items[0].metadata.name}') -n microservices-demo -- /bin/sh
```

Inside the frontend pod, try to access the backend service:

```bash
curl http://backend:80
```

You should get a successful response (like the default HTML page from the httpd container).

#### 3.2 Check Frontend to Database Communication

While still in the frontend pod, try to access the database service:

```bash
curl http://database:8080
```

You should receive a connection error, indicating that the frontend cannot access the database.

### Conclusion

You’ve successfully set up a microservices architecture in Kubernetes with network policies controlling communication between the frontend, backend, and database services. This implementation ensures that:

- The frontend can communicate with the backend.
- The frontend cannot access the database directly.

Feel free to customize the services or network policies further based on your requirements. If you have any questions or need additional features, let me know!
