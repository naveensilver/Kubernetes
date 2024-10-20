Let's implement SSL termination using Kubernetes Ingress to manage HTTPS traffic centrally for a financial application. This example will demonstrate how to set up Ingress with SSL certificates to handle secure connections.

### Use Case: SSL Termination for a Financial Application

We'll set up an Ingress resource that terminates SSL connections, allowing us to route HTTP traffic to various microservices after handling the SSL encryption.

### Step-by-Step Implementation

#### Step 1: Set Up Kubernetes Cluster

If you haven't set up a Kubernetes cluster yet, follow the initial steps from the previous implementation.

1. **Start Minikube:**

   ```bash
   minikube start
   ```

2. **Enable the Ingress Add-on:**

   ```bash
   minikube addons enable ingress
   ```

#### Step 2: Create Namespace

Create a dedicated namespace for the application:

```bash
kubectl create namespace finance-app
```

#### Step 3: Deploy Sample Microservices

We'll create two sample microservices for this example: a **User Service** and a **Transaction Service**.

##### 1. User Service

Create a file named `user-service.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: finance-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user
  template:
    metadata:
      labels:
        app: user
    spec:
      containers:
      - name: user
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=User Service"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: finance-app
spec:
  type: ClusterIP
  ports:
  - port: 5678
    targetPort: 5678
  selector:
    app: user
```

##### 2. Transaction Service

Create a file named `transaction-service.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: transaction-service
  namespace: finance-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: transaction
  template:
    metadata:
      labels:
        app: transaction
    spec:
      containers:
      - name: transaction
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=Transaction Service"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: transaction-service
  namespace: finance-app
spec:
  type: ClusterIP
  ports:
  - port: 5678
    targetPort: 5678
  selector:
    app: transaction
```

#### Step 4: Deploy the Services

Run the following commands to create the services and deployments in the `finance-app` namespace:

```bash
kubectl apply -f user-service.yaml
kubectl apply -f transaction-service.yaml
```

#### Step 5: Generate SSL Certificates

For SSL termination, you need an SSL certificate and a key. For testing purposes, you can create a self-signed certificate. Use OpenSSL:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=finance-app.local/O=My Company"
```

This will create two files: `tls.crt` (certificate) and `tls.key` (private key).

#### Step 6: Create a Secret for SSL Certificates

Create a Kubernetes secret to store the SSL certificate and key:

```bash
kubectl create secret tls finance-app-tls --cert=tls.crt --key=tls.key -n finance-app
```

#### Step 7: Create Ingress Resource

Create a file named `ingress.yaml` to define the Ingress resource with SSL termination:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: finance-app-ingress
  namespace: finance-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - finance-app.local
    secretName: finance-app-tls
  rules:
  - host: finance-app.local
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 5678
      - path: /transactions
        pathType: Prefix
        backend:
          service:
            name: transaction-service
            port:
              number: 5678
```

#### Step 8: Apply the Ingress Resource

Run the following command to create the Ingress resource:

```bash
kubectl apply -f ingress.yaml
```

#### Step 9: Update Local Hosts File

To access the services using the defined hostname, update your local `/etc/hosts` file (or `C:\Windows\System32\drivers\etc\hosts` on Windows) to point `finance-app.local` to your Minikube IP. Get the Minikube IP:

```bash
minikube ip
```

Add the following line to your hosts file (replace `<MINIKUBE-IP>` with the actual IP address):

```
<MINIKUBE-IP> finance-app.local
```

#### Step 10: Access the Services

You can now access the services using HTTPS:

- **User Service**: [https://finance-app.local/users](https://finance-app.local/users)
- **Transaction Service**: [https://finance-app.local/transactions](https://finance-app.local/transactions)

Since you are using a self-signed certificate, your browser will likely show a security warning. You can proceed past this warning to see the respective messages for each service.

#### Step 11: Monitoring the Deployment

You can check the status of your deployments and services using:

```bash
kubectl get deployments -n finance-app
kubectl get services -n finance-app
kubectl get ingress -n finance-app
```

#### Step 12: Cleanup Resources

To delete the Ingress and applications, run:

```bash
kubectl delete -f ingress.yaml
kubectl delete -f user-service.yaml
kubectl delete -f transaction-service.yaml
kubectl delete secret finance-app-tls -n finance-app
```

### Conclusion

In this implementation:
- We set up a Kubernetes cluster with an Ingress controller.
- We deployed two microservices for the financial application.
- We generated SSL certificates and created a Kubernetes secret for SSL termination.
- An Ingress resource was created to manage HTTPS traffic and route requests to the respective microservices.

This setup demonstrates how to manage SSL termination centrally using Ingress, simplifying certificate management and offloading SSL processing from individual services.

If you have any further questions or need additional features, feel free to ask!
