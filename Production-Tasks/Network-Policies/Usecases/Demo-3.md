Let’s walk through a detailed implementation of a database access control scenario in Kubernetes. We will set up a simple application architecture where only specific application pods are allowed to access a database pod. We will achieve this using Kubernetes network policies to control egress traffic.

### Step-by-Step Implementation

#### Step 1: Set Up the Kubernetes Environment

Ensure you have a running Kubernetes cluster and `kubectl` configured to access it.

### Step 2: Create Namespaces

We’ll create a namespace for our application to keep things organized.

```bash
kubectl create namespace app-demo
```

### Step 3: Deploy the Database Service

We’ll deploy a simple database (e.g., a PostgreSQL database) along with its service.

**database-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: app-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_USER
          value: "user"
        - name: POSTGRES_PASSWORD
          value: "password"
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: app-demo
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

Apply the database deployment:

```bash
kubectl apply -f database-deployment.yaml
```

### Step 4: Deploy the Application Pods

Now, let’s create two different application pods: one that will be allowed to access the database and another that will not.

**allowed-app-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: allowed-app
  namespace: app-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: allowed-app
  template:
    metadata:
      labels:
        app: allowed-app
    spec:
      containers:
      - name: allowed-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "tail -f /dev/null"]
```

**blocked-app-deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blocked-app
  namespace: app-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blocked-app
  template:
    metadata:
      labels:
        app: blocked-app
    spec:
      containers:
      - name: blocked-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c", "tail -f /dev/null"]
```

Apply the application deployments:

```bash
kubectl apply -f allowed-app-deployment.yaml
kubectl apply -f blocked-app-deployment.yaml
```

### Step 5: Create the Egress Network Policy

Now, we will create an egress network policy that allows only the `allowed-app` to access the database.

**egress-policy.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-access
  namespace: app-demo
spec:
  podSelector:
    matchLabels:
      app: allowed-app
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
```

Apply the egress network policy:

```bash
kubectl apply -f egress-policy.yaml
```

### Step 6: Testing the Access Control

To verify that our configuration works as expected, we’ll attempt to access the database from both application pods.

#### 6.1 Get Pod Names

First, get the names of the pods for reference.

```bash
# Allowed App Pod
kubectl get pods -n app-demo -l app=allowed-app

# Blocked App Pod
kubectl get pods -n app-demo -l app=blocked-app
```

#### 6.2 Access Database from Allowed Application

Start a shell in the allowed application pod:

```bash
kubectl exec -it $(kubectl get pods -n app-demo -l app=allowed-app -o jsonpath='{.items[0].metadata.name}') -n app-demo -- /bin/sh
```

Inside the allowed application pod, try to access the database:

```bash
apt-get update && apt-get install -y postgresql-client
PGPASSWORD=password psql -h postgres -U user -c '\l'
```

You should see a list of databases, confirming that the allowed application can access the database.

#### 6.3 Access Database from Blocked Application

Now, start a shell in the blocked application pod:

```bash
kubectl exec -it $(kubectl get pods -n app-demo -l app=blocked-app -o jsonpath='{.items[0].metadata.name}') -n app-demo -- /bin/sh
```

Inside the blocked application pod, try to access the database:

```bash
apt-get update && apt-get install -y postgresql-client
PGPASSWORD=password psql -h postgres -U user -c '\l'
```

You should receive a connection error, indicating that the blocked application cannot access the database.

### Conclusion

You have successfully implemented database access control in Kubernetes using network policies. The `allowed-app` can access the database, while the `blocked-app` cannot. This approach helps maintain security by ensuring only authorized applications can interact with sensitive resources like databases.

If you have any further questions or want to explore more advanced scenarios, feel free to ask!
