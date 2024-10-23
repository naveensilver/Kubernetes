Let’s walk through the implementation of an egress network policy that restricts access to an external API. In this scenario, only specific pods labeled with `role=api-client` will be allowed to access the external API, while other pods will be denied access.

### Step-by-Step Implementation

#### Step 1: Set Up the Kubernetes Environment

Ensure you have a running Kubernetes cluster and `kubectl` configured to access it.

### Step 2: Create a Namespace

We’ll create a namespace for our application.

```bash
kubectl create namespace api-demo
```

### Step 3: Deploy Pods

We will deploy two types of application pods: one set that will be allowed to access the external API and another set that will not.

#### 3.1 Deploy Allowed Pods

**allowed-api-client-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-client
  namespace: api-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      role: api-client
  template:
    metadata:
      labels:
        role: api-client
    spec:
      containers:
      - name: api-client
        image: curlimages/curl:latest
        command: ["/bin/sh", "-c", "tail -f /dev/null"]  # Keep the container running
```

#### 3.2 Deploy Blocked Pods

**blocked-api-client-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-api-client
  namespace: api-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      role: no-api-client
  template:
    metadata:
      labels:
        role: no-api-client
    spec:
      containers:
      - name: no-api-client
        image: curlimages/curl:latest
        command: ["/bin/sh", "-c", "tail -f /dev/null"]  # Keep the container running
```

Apply both deployments:

```bash
kubectl apply -f allowed-api-client-deployment.yaml
kubectl apply -f blocked-api-client-deployment.yaml
```

### Step 4: Create the Egress Network Policy

Next, we’ll create an egress network policy that allows only the pods labeled with `role=api-client` to access an external API (for demonstration purposes, we will use `http://httpbin.org/get`).

**egress-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api-access
  namespace: api-demo
spec:
  podSelector:
    matchLabels:
      role: api-client
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 104.18.12.0/22  # This is the CIDR for httpbin.org (example)
```

Apply the network policy:

```bash
kubectl apply -f egress-policy.yaml
```

### Step 5: Testing the Access Control

To verify that our configuration works as expected, we’ll attempt to access the external API from both sets of pods.

#### 5.1 Get Pod Names

First, get the names of the pods for reference.

```bash
# Allowed API Client Pod
kubectl get pods -n api-demo -l role=api-client

# Blocked API Client Pod
kubectl get pods -n api-demo -l role=no-api-client
```

#### 5.2 Access External API from Allowed Pod

Start a shell in the allowed API client pod:

```bash
kubectl exec -it $(kubectl get pods -n api-demo -l role=api-client -o jsonpath='{.items[0].metadata.name}') -n api-demo -- /bin/sh
```

Inside the allowed pod, try to access the external API:

```bash
curl http://httpbin.org/get
```

You should receive a valid response from the external API, confirming that the allowed pod can access it.

#### 5.3 Access External API from Blocked Pod

Now, start a shell in the blocked API client pod:

```bash
kubectl exec -it $(kubectl get pods -n api-demo -l role=no-api-client -o jsonpath='{.items[0].metadata.name}') -n api-demo -- /bin/sh
```

Inside the blocked pod, try to access the external API:

```bash
curl http://httpbin.org/get
```

You should receive a connection error, indicating that the blocked pod cannot access the external API.

### Conclusion

You have successfully implemented an egress network policy that restricts access to an external API. Only the pods labeled with `role=api-client` can reach the external API, while other pods are denied access. This approach helps maintain security by controlling which pods can interact with external services.

