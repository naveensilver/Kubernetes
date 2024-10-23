Integrating a service mesh like Istio with Kubernetes network policies allows you to enhance security by controlling traffic between services. In this scenario, we'll implement network policies that restrict communication between specific pods within the Istio mesh while still utilizing Istio's service discovery features.

### Step-by-Step Implementation

#### Step 1: Set Up the Kubernetes Environment

Ensure you have a running Kubernetes cluster with Istio installed. You can follow the official Istio documentation for installation if it's not already set up.

### Step 2: Create Namespaces

We’ll create two namespaces to simulate a multi-service environment within the service mesh. For this example, we'll use `frontend` and `backend`.

```bash
kubectl create namespace frontend
kubectl create namespace backend
```

### Step 3: Deploy Sample Applications

Let’s deploy a simple frontend service and a backend service in their respective namespaces.

#### 3.1 Deploy the Backend Service

**backend-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        istio: backend
    spec:
      containers:
      - name: backend
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "echo 'Backend Service' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]
```

Apply the backend deployment:

```bash
kubectl apply -f backend-deployment.yaml
```

#### 3.2 Deploy the Frontend Service

**frontend-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        istio: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "tail -f /dev/null"]  # Keep the container running
```

Apply the frontend deployment:

```bash
kubectl apply -f frontend-deployment.yaml
```

### Step 4: Create Network Policies

Now, we’ll create network policies to control traffic between the frontend and backend services.

#### 4.1 Network Policy for Backend Service

This policy allows traffic to the backend pods only from the frontend pods.

**backend-ingress-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-access
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          istio-injection: enabled
      ports:
      - protocol: TCP
        port: 80
```

Apply the ingress network policy for the backend:

```bash
kubectl apply -f backend-ingress-policy.yaml
```

#### 4.2 Network Policy for Frontend Service

You can also create a network policy for the frontend service to restrict communication to only specific external services if needed, but for this scenario, we'll focus on backend access.

### Step 5: Testing the Access Control

To verify that the configuration works as expected, we’ll check communication between the frontend and backend services.

#### 5.1 Get Pod Names

Get the names of the frontend and backend pods for reference.

```bash
# Frontend Pod
kubectl get pods -n frontend

# Backend Pod
kubectl get pods -n backend
```

#### 5.2 Test Communication from Frontend to Backend

Start a shell in the frontend pod:

```bash
kubectl exec -it $(kubectl get pods -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}') -n frontend -- /bin/sh
```

Inside the frontend pod, try to access the backend service:

```bash
curl http://backend.backend.svc.cluster.local
```

You should receive a response from the backend service, confirming that the frontend can access it.

#### 5.3 Test Communication from Other Sources

To ensure that the backend is not accessible from other sources, you can deploy a simple busybox pod in a different namespace.

**busybox-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "tail -f /dev/null"]  # Keep the container running
```

Apply the busybox deployment:

```bash
kubectl apply -f busybox-deployment.yaml
```

Now, start a shell in the busybox pod:

```bash
kubectl exec -it $(kubectl get pods -n default -l app=busybox -o jsonpath='{.items[0].metadata.name}') -n default -- /bin/sh
```

Inside the busybox pod, try to access the backend service:

```bash
curl http://backend.backend.svc.cluster.local
```

You should receive a connection error, confirming that the backend service is not accessible from the busybox pod in the default namespace.

### Conclusion

You have successfully implemented network policies in a service mesh environment to control traffic between services. The backend service is now only accessible by the frontend service, enhancing security while maintaining the service discovery capabilities provided by Istio.

If you have further questions or want to explore more advanced scenarios, feel free to ask!
