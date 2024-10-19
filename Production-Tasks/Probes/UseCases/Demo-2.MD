Let’s implement an end-to-end example of a stateful application using PostgreSQL in Kubernetes, focusing on liveness and readiness probes to ensure the database is ready to handle requests.

### Scenario Overview

We'll create a PostgreSQL database deployment with proper probes to ensure it's healthy and ready to accept connections. This setup will demonstrate how Kubernetes can manage the lifecycle of stateful applications.

### Step 1: Create the PostgreSQL Deployment

1. **Create a Namespace** (optional):
   ```bash
   kubectl create namespace postgres-demo
   ```

2. **Create a Persistent Volume Claim**: This is necessary for PostgreSQL to store its data persistently.

   Create a file named `postgres-pvc.yaml`:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: postgres-pvc
     namespace: postgres-demo
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
   ```

   Apply the PVC:
   ```bash
   kubectl apply -f postgres-pvc.yaml
   ```

3. **Create a Deployment Manifest**: Create a file named `postgres-deployment.yaml`.

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: postgres
     namespace: postgres-demo
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
           image: postgres:14
           env:
           - name: POSTGRES_USER
             value: "admin"
           - name: POSTGRES_PASSWORD
             value: "password"
           - name: POSTGRES_DB
             value: "mydatabase"
           ports:
           - containerPort: 5432
           volumeMounts:
           - name: postgres-storage
             mountPath: /var/lib/postgresql/data
           livenessProbe:
             exec:
               command:
                 - "pg_isready"
                 - "-U"
                 - "admin"
             initialDelaySeconds: 30
             periodSeconds: 10
           readinessProbe:
             exec:
               command:
                 - "pg_isready"
                 - "-U"
                 - "admin"
             initialDelaySeconds: 10
             periodSeconds: 5
         volumes:
         - name: postgres-storage
           persistentVolumeClaim:
             claimName: postgres-pvc
   ```

### Step 2: Apply the Deployment

1. **Deploy PostgreSQL**:
   ```bash
   kubectl apply -f postgres-deployment.yaml
   ```

### Step 3: Verify the Deployment

1. **Check the Pods**:
   ```bash
   kubectl get pods -n postgres-demo
   ```

   You should see the PostgreSQL Pod running.

2. **Check the Logs** (optional):
   ```bash
   kubectl logs <pod-name> -n postgres-demo
   ```

### Step 4: Test the Probes

1. **Get the Pod IP**:
   ```bash
   kubectl get pods -o wide -n postgres-demo
   ```

2. **Use `kubectl port-forward` to Access PostgreSQL**:
   ```bash
   kubectl port-forward <pod-name> 5432:5432 -n postgres-demo
   ```

3. **Test Readiness**:
   Use `pg_isready` to check if the database is ready:
   ```bash
   psql -h localhost -U admin -d mydatabase -c "SELECT 'Connection successful';"
   ```

   If the readiness probe is working, you should see the connection message. If not, it might indicate the database is not ready.

### Step 5: Clean Up

1. **Delete the Deployment and PVC**:
   ```bash
   kubectl delete -f postgres-deployment.yaml
   kubectl delete -f postgres-pvc.yaml
   ```

### Explanation of Key Components

- **Persistent Volume Claim (PVC)**: Ensures that the PostgreSQL data persists even if the Pod is restarted or rescheduled.
- **Liveness Probe**: Uses `pg_isready` to check if the database is alive. If the check fails, Kubernetes will restart the Pod.
- **Readiness Probe**: Also uses `pg_isready` but checks if the database is ready to accept connections. It ensures that traffic is only sent to Pods that can handle requests.
- **Environment Variables**: Used to set up the database user, password, and initial database.

### Conclusion

In this hands-on implementation, we:
- Deployed a PostgreSQL database on Kubernetes with persistent storage.
- Configured liveness and readiness probes to ensure the database is healthy and ready to accept connections.

This setup is critical for managing stateful applications effectively in a Kubernetes environment. If you have any questions or need further details, feel free to ask!
