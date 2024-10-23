To implement an end-to-end solution that prevents development pods from accessing production resources using Kubernetes **Network Policies**, we will:

1. **Set up namespaces for development and production**.
2. **Deploy sample pods and services** in both environments.
3. **Create Network Policies** to isolate the development namespace from production resources.
4. **Allow only specific CI/CD tools to interact with production**.

Here’s how you can implement this scenario step-by-step.

### Step 1: Create Namespaces for Development and Production
We will create two namespaces: `development` for testing and `production` for actual production workloads.

```bash
# Create namespaces
kubectl create namespace development
kubectl create namespace production
```

### Step 2: Deploy Sample Pods and Services

We'll deploy some example pods and services in both the `development` and `production` namespaces.

#### 2.1 Deploy a Sample Pod in the Production Namespace
Let’s deploy an NGINX pod in the `production` namespace, representing a production service.

```yaml
# production-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-nginx
  labels:
    app: nginx
  namespace: production
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: production-service
  namespace: production
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Deploy the production pod and service:

```bash
kubectl apply -f production-pod.yaml
```

#### 2.2 Deploy a Sample Pod in the Development Namespace
Similarly, deploy a sample NGINX pod in the `development` namespace, representing a testing service.

```yaml
# development-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dev-nginx
  labels:
    app: nginx
  namespace: development
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

Deploy the development pod:

```bash
kubectl apply -f development-pod.yaml
```

### Step 3: Create Network Policies to Block Development Access to Production

Next, we create a **Network Policy** that blocks any pod in the `development` namespace from accessing resources in the `production` namespace. 

#### 3.1 Create the Default Deny Policy for Production
This policy blocks **all** ingress traffic to the production pods by default.

```yaml
# deny-all-production.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-production
  namespace: production
spec:
  podSelector: {}  # Select all pods in the production namespace
  policyTypes:
  - Ingress
```

Deploy the policy:

```bash
kubectl apply -f deny-all-production.yaml
```

#### 3.2 Allow Specific CI/CD Tools to Access Production
Let’s say we have a CI/CD tool pod labeled as `ci-cd: true` in the `development` namespace that needs to deploy to production. We create a policy to allow traffic from that pod only.

```yaml
# allow-ci-cd-access.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ci-cd-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: nginx  # Only allow access to nginx pods in production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: development  # From the development namespace
      podSelector:
        matchLabels:
          ci-cd: true  # But only from pods labeled ci-cd: true
```

This policy allows access to production resources only from a specific pod in the development namespace.

Deploy the policy:

```bash
kubectl apply -f allow-ci-cd-access.yaml
```

### Step 4: Test the Setup

#### 4.1 Access Production Service from Development Pod
To test whether access to the production environment is blocked, we’ll try to access the `production-nginx` pod from the `development` namespace.

```bash
# Start an interactive shell inside the development pod
kubectl exec -it dev-nginx -n development -- /bin/bash

# Try to curl the production service (replace CLUSTER-IP with actual service IP)
curl <PRODUCTION-SERVICE-CLUSTER-IP>:80
```

You should see that the request is **blocked**.

#### 4.2 Test Access from a CI/CD Tool Pod
Now, let’s deploy a dummy CI/CD tool pod labeled as `ci-cd: true` in the `development` namespace and test access.

```yaml
# ci-cd-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ci-cd-tool
  labels:
    ci-cd: true
  namespace: development
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
```

Deploy the pod:

```bash
kubectl apply -f ci-cd-pod.yaml
```

Once deployed, exec into the CI/CD pod and test access to the production service:

```bash
kubectl exec -it ci-cd-tool -n development -- sh

# Try to curl the production service (replace CLUSTER-IP with actual service IP)
curl <PRODUCTION-SERVICE-CLUSTER-IP>:80
```

The request should be **successful** from this pod, while it remains blocked from other pods in the development namespace.

### Step 5: Monitoring and Logging

- **Monitor Network Policies**: Use tools like **Cilium** or **Calico** for monitoring network traffic and policy enforcement. These tools provide visibility into how traffic is being allowed or denied according to your policies.
  
- **Logging Policy Events**: Enable logging to ensure you can track allowed and denied requests, making debugging easier.

---

### Conclusion:
This end-to-end implementation demonstrates how to:

- Create separate namespaces for `development` and `production`.
- Block access between development and production pods using Network Policies.
- Allow specific CI/CD tools to interact with production resources while preventing other development pods from accessing them.

This approach ensures strict traffic isolation in production environments and enforces security best practices in multi-tenant Kubernetes environments.
