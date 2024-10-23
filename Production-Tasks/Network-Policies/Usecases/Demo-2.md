To isolate development and production environments in Kubernetes using network policies, we’ll set up two separate namespaces: `development` and `production`. The goal is to prevent any pods in the `development` namespace from accessing resources in the `production` namespace.

### Step-by-Step Implementation

#### Step 1: Create Namespaces

First, create the `development` and `production` namespaces.

```bash
kubectl create namespace development
kubectl create namespace production
```

#### Step 2: Deploy Example Applications

For this example, we’ll deploy a simple application in each namespace.

##### 2.1 Deploy in the Production Namespace

**production-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
    spec:
      containers:
      - name: production-app
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: production-app
  namespace: production
spec:
  selector:
    app: production-app
  ports:
  - port: 80
    targetPort: 80
```

Apply the production deployment:

```bash
kubectl apply -f production-deployment.yaml
```

##### 2.2 Deploy in the Development Namespace

**development-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: development-app
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: development-app
  template:
    metadata:
      labels:
        app: development-app
    spec:
      containers:
      - name: development-app
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: development-app
  namespace: development
spec:
  selector:
    app: development-app
  ports:
  - port: 80
    targetPort: 80
```

Apply the development deployment:

```bash
kubectl apply -f development-deployment.yaml
```

#### Step 3: Create Network Policies

Now, let’s create a network policy in the `production` namespace that denies all ingress traffic from the `development` namespace.

##### 3.1 Deny Traffic from Development Namespace

**deny-development-ingress.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-development-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: development
```

Apply the network policy:

```bash
kubectl apply -f deny-development-ingress.yaml
```

### Step 4: Testing the Network Policies

To test the network policies, we will attempt to access the production application from the development application.

#### 4.1 Get the Pod Names

First, get the names of the pods in both namespaces.

```bash
# Production Pod
kubectl get pods -n production

# Development Pod
kubectl get pods -n development
```

#### 4.2 Access Production from Development

Start a shell in the development pod:

```bash
kubectl exec -it $(kubectl get pods -n development -l app=development-app -o jsonpath='{.items[0].metadata.name}') -n development -- /bin/sh
```

Inside the development pod, try to access the production service:

```bash
curl http://production-app:80
```

You should receive a connection error, indicating that the development pod cannot access the production application.

### Step 5: Allow Specific Access (Optional)

If you later decide to allow specific access (e.g., from a monitoring service), you can create additional network policies in the production namespace to allow that traffic.

To allow specific access to your production resources while maintaining the isolation between development and production namespaces, you can create additional network policies. For instance, let’s say you want to allow access from a monitoring service running in the development namespace.

### Step-5:  Step-by-Step Implementation to Allow Specific Access

#### Step 1: Deploy a Monitoring Service

Let’s create a simple monitoring service in the development namespace that needs access to the production application. This could be a basic HTTP monitoring tool.

**monitoring-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-app
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monitoring-app
  template:
    metadata:
      labels:
        app: monitoring-app
    spec:
      containers:
      - name: monitoring-app
        image: nginx:alpine
        ports:
        - containerPort: 80
```

Apply the monitoring deployment:

```bash
kubectl apply -f monitoring-deployment.yaml
```

#### Step 2: Create a Network Policy to Allow Access

Next, we’ll create a network policy in the `production` namespace that allows traffic from the `monitoring-app` in the `development` namespace.

**allow-monitoring-access.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: production-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: development
      # You can also specify podSelector here if you want to restrict further
      # Uncomment the following lines to restrict to a specific pod
      # podSelector:
      #   matchLabels:
      #     app: monitoring-app
```

Apply the network policy:

```bash
kubectl apply -f allow-monitoring-access.yaml
```

### Step 3: Testing the Access

#### 3.1 Access Production from Monitoring Service

To test that the monitoring service can access the production application, start a shell in the monitoring pod:

```bash
kubectl exec -it $(kubectl get pods -n development -l app=monitoring-app -o jsonpath='{.items[0].metadata.name}') -n development -- /bin/sh
```

Inside the monitoring pod, try to access the production service:

```bash
curl http://production-app:80
```

You should receive a successful response, confirming that the monitoring service can access the production application.

### Conclusion

By implementing this additional network policy, you’ve allowed specific access from the monitoring service in the development namespace to the production application while still preventing other traffic from the development namespace. This allows you to maintain security while enabling necessary monitoring or management functionalities.


