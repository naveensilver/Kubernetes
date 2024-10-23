To restrict access to an API gateway so that it is only accessible by certain internal services or namespaces, we can create an ingress network policy. This policy will define which internal services are allowed to communicate with the API gateway, enhancing security by preventing unauthorized access.

### Step-by-Step Implementation

#### Step 1: Set Up the Kubernetes Environment

Make sure you have a running Kubernetes cluster with `kubectl` configured to access it.

### Step 2: Create Namespaces

We'll create two namespaces: one for the internal services and another for the API gateway.

```bash
kubectl create namespace api-gateway
kubectl create namespace internal-services
```

### Step 3: Deploy the API Gateway

We'll deploy a simple API gateway service. For this example, we’ll use Nginx to represent the API gateway.

**api-gateway-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: api-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "echo 'API Gateway' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]
```

Apply the API gateway deployment:

```bash
kubectl apply -f api-gateway-deployment.yaml
```

### Step 4: Deploy Internal Services

Next, we’ll deploy some internal services that will be allowed to access the API gateway.

**internal-service-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: internal-service
  namespace: internal-services
spec:
  replicas: 1
  selector:
    matchLabels:
      app: internal-service
  template:
    metadata:
      labels:
        app: internal-service
    spec:
      containers:
      - name: internal-service
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "tail -f /dev/null"]  # Keep the container running
```

Apply the internal service deployment:

```bash
kubectl apply -f internal-service-deployment.yaml
```

### Step 5: Create the Ingress Network Policy

Now, we’ll create an ingress network policy for the API gateway that allows traffic only from the internal service namespace.

**api-gateway-ingress-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-access
  namespace: api-gateway
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: internal-services
```

Apply the ingress network policy for the API gateway:

```bash
kubectl apply -f api-gateway-ingress-policy.yaml
```

### Step 6: Testing the Access Control

To verify that the configuration works as expected, we’ll check the communication between the internal service and the API gateway.

#### 6.1 Get Pod Names

First, get the names of the API gateway and internal service pods for reference.

```bash
# API Gateway Pod
kubectl get pods -n api-gateway

# Internal Service Pod
kubectl get pods -n internal-services
```

#### 6.2 Test Communication from Internal Service to API Gateway

Start a shell in the internal service pod:

```bash
kubectl exec -it $(kubectl get pods -n internal-services -l app=internal-service -o jsonpath='{.items[0].metadata.name}') -n internal-services -- /bin/sh
```

Inside the internal service pod, try to access the API gateway:

```bash
curl http://api-gateway.api-gateway.svc.cluster.local
```

You should receive a response from the API gateway, confirming that the internal service can access it.

#### 6.3 Test Communication from Other Sources

To ensure that the API gateway is not accessible from other sources, you can deploy a simple busybox pod in a different namespace.

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

Inside the busybox pod, try to access the API gateway:

```bash
curl http://api-gateway.api-gateway.svc.cluster.local
```

You should receive a connection error, confirming that the API gateway is not accessible from the busybox pod in the default namespace.

### Conclusion

You have successfully implemented an ingress network policy that restricts access to the API gateway, allowing traffic only from specific internal services. This setup enhances security by ensuring that only authorized internal services can communicate with the API gateway.

If you have any further questions or want to explore additional scenarios, feel free to ask!
