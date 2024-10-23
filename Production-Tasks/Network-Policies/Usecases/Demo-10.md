To restrict access to monitoring and logging services in Kubernetes, we can implement network policies that limit ingress traffic to these services, allowing only authorized application pods to send data to them. This enhances security and ensures that only specific applications can communicate with your monitoring and logging infrastructure.

### Step-by-Step Implementation

#### Step 1: Set Up the Kubernetes Environment

Make sure you have a running Kubernetes cluster with `kubectl` configured to access it.

### Step 2: Create Namespaces

We'll create two namespaces: one for the monitoring and logging services and another for the application pods.

```bash
kubectl create namespace monitoring
kubectl create namespace applications
```

### Step 3: Deploy Monitoring and Logging Services

For this example, let's simulate a monitoring service using a simple Nginx pod. You can replace it with a real monitoring service like Prometheus or Grafana later.

**monitoring-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      containers:
      - name: monitoring
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "echo 'Monitoring Service' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]
```

Apply the monitoring service deployment:

```bash
kubectl apply -f monitoring-deployment.yaml
```

### Step 4: Deploy Authorized Application Pods

Now, let's deploy an application pod that will be allowed to access the monitoring service.

**authorized-app-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authorized-app
  namespace: applications
spec:
  replicas: 1
  selector:
    matchLabels:
      app: authorized-app
  template:
    metadata:
      labels:
        app: authorized-app
    spec:
      containers:
      - name: authorized-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "tail -f /dev/null"]  # Keep the container running
```

Apply the authorized application deployment:

```bash
kubectl apply -f authorized-app-deployment.yaml
```

### Step 5: Deploy an Unauthorized Application Pod

For testing purposes, let's also deploy a pod that will not be allowed to access the monitoring service.

**unauthorized-app-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unauthorized-app
  namespace: applications
spec:
  replicas: 1
  selector:
    matchLabels:
      app: unauthorized-app
  template:
    metadata:
      labels:
        app: unauthorized-app
    spec:
      containers:
      - name: unauthorized-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "tail -f /dev/null"]  # Keep the container running
```

Apply the unauthorized application deployment:

```bash
kubectl apply -f unauthorized-app-deployment.yaml
```

### Step 6: Create Ingress Network Policy for Monitoring Service

Now, we’ll create a network policy that allows ingress traffic to the monitoring service only from the authorized application pods.

**monitoring-ingress-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-authorized-app-access
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app: monitoring
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: authorized-app
```

Apply the ingress network policy for the monitoring service:

```bash
kubectl apply -f monitoring-ingress-policy.yaml
```

### Step 7: Testing the Access Control

To verify that the configuration works as expected, we’ll check communication from both the authorized and unauthorized application pods to the monitoring service.

#### 7.1 Get Pod Names

First, get the names of the monitoring and application pods for reference.

```bash
# Monitoring Pod
kubectl get pods -n monitoring

# Authorized Application Pod
kubectl get pods -n applications -l app=authorized-app

# Unauthorized Application Pod
kubectl get pods -n applications -l app=unauthorized-app
```

#### 7.2 Test Communication from Authorized Application to Monitoring Service

Start a shell in the authorized application pod:

```bash
kubectl exec -it $(kubectl get pods -n applications -l app=authorized-app -o jsonpath='{.items[0].metadata.name}') -n applications -- /bin/sh
```

Inside the authorized application pod, try to access the monitoring service:

```bash
curl http://monitoring.monitoring.svc.cluster.local
```

You should receive a response from the monitoring service, confirming that the authorized application can access it.

#### 7.3 Test Communication from Unauthorized Application to Monitoring Service

Now, start a shell in the unauthorized application pod:

```bash
kubectl exec -it $(kubectl get pods -n applications -l app=unauthorized-app -o jsonpath='{.items[0].metadata.name}') -n applications -- /bin/sh
```

Inside the unauthorized application pod, try to access the monitoring service:

```bash
curl http://monitoring.monitoring.svc.cluster.local
```

You should receive a connection error, confirming that the unauthorized application cannot access the monitoring service.

### Conclusion

You have successfully implemented an ingress network policy that restricts access to the monitoring service, allowing traffic only from specific authorized application pods. This setup enhances security by ensuring that only designated services can send data to your monitoring and logging infrastructure.

If you have further questions or want to explore additional scenarios, feel free to ask!
