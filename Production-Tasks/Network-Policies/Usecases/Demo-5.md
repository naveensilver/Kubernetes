To implement a multi-tenancy scenario in Kubernetes where multiple teams are using the same cluster, we can create network policies to ensure that each team’s workloads are isolated. This involves setting up separate namespaces for each team and defining network policies that restrict communication between pods in different namespaces.

### Step-by-Step Implementation

#### Step 1: Set Up the Kubernetes Environment

Make sure you have a running Kubernetes cluster and `kubectl` configured to access it.

### Step 2: Create Namespaces for Each Team

We’ll create two namespaces: one for Team A and one for Team B.

```bash
kubectl create namespace team-a
kubectl create namespace team-b
```

### Step 3: Deploy Sample Applications

We will deploy simple applications in both namespaces. Each team will have a deployment with multiple pods.

#### 3.1 Deploy Applications for Team A

**team-a-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: team-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-a
  template:
    metadata:
      labels:
        app: app-a
    spec:
      containers:
      - name: app-a
        image: nginx:alpine
        ports:
        - containerPort: 80
```

Apply the deployment for Team A:

```bash
kubectl apply -f team-a-deployment.yaml
```

#### 3.2 Deploy Applications for Team B

**team-b-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
  namespace: team-b
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-b
  template:
    metadata:
      labels:
        app: app-b
    spec:
      containers:
      - name: app-b
        image: nginx:alpine
        ports:
        - containerPort: 80
```

Apply the deployment for Team B:

```bash
kubectl apply -f team-b-deployment.yaml
```

### Step 4: Create Network Policies

Now we’ll create network policies for each team’s namespace to restrict traffic.

#### 4.1 Network Policy for Team A

This policy allows traffic only between pods in Team A's namespace.

**team-a-network-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-team
  namespace: team-a
spec:
  podSelector:
    matchLabels:
      app: app-a
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: app-a
```

Apply the network policy for Team A:

```bash
kubectl apply -f team-a-network-policy.yaml
```

#### 4.2 Network Policy for Team B

This policy allows traffic only between pods in Team B's namespace.

**team-b-network-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-team
  namespace: team-b
spec:
  podSelector:
    matchLabels:
      app: app-b
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: app-b
```

Apply the network policy for Team B:

```bash
kubectl apply -f team-b-network-policy.yaml
```

### Step 5: Testing the Isolation

To verify that the isolation works, we’ll check the communication between the pods of both teams.

#### 5.1 Get Pod Names

First, get the names of the pods for reference.

```bash
# Team A Pods
kubectl get pods -n team-a

# Team B Pods
kubectl get pods -n team-b
```

#### 5.2 Test Communication Within Team A

Start a shell in one of the Team A pods:

```bash
kubectl exec -it $(kubectl get pods -n team-a -l app=app-a -o jsonpath='{.items[0].metadata.name}') -n team-a -- /bin/sh
```

Inside the Team A pod, try to access another Team A pod:

```bash
curl http://app-a:80
```

You should receive a successful response.

#### 5.3 Test Communication Between Teams

Now, start a shell in one of the Team B pods:

```bash
kubectl exec -it $(kubectl get pods -n team-b -l app=app-b -o jsonpath='{.items[0].metadata.name}') -n team-b -- /bin/sh
```

Inside the Team B pod, try to access a Team A pod:

```bash
curl http://<team-a-pod-ip>:80
```

You should receive a connection error, confirming that communication between teams is blocked.

### Conclusion

You have successfully implemented multi-tenancy in Kubernetes by isolating workloads between different teams. Each team can communicate within their own namespace, but communication between different team namespaces is restricted. This approach enhances security and resource management in a shared Kubernetes environment.

If you have any further questions or want to explore additional features, feel free to ask!
