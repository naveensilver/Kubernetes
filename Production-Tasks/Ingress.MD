# Securing the Communication Between Users and Services with Ingress

Step-by-step production-level implementation of securing communication using an Ingress controller and TLS certificates with Cert-Manager and Let's Encrypt** in Kubernetes. This guide will also include proper examples for a real-world use case where we expose an NGINX application via Ingress with HTTPS encryption.

### **Objective**

We aim to:
1. Deploy an **NGINX** application.
2. Set up an **Ingress** to route traffic to this application.
3. Secure communication using **Cert-Manager** and **Let's Encrypt** for automatic SSL/TLS certificates.

### **Prerequisites**

1. **Kubernetes Cluster**: A running Kubernetes cluster (on a cloud provider like AWS EKS, GCP GKE, Azure AKS, or a self-hosted cluster).
2. **kubectl**: Installed and configured to interact with the cluster.
3. **NGINX Ingress Controller**: Installed in your Kubernetes cluster.
4. **Domain Name**: A registered domain name (e.g., `example.com`) pointing to the cluster's IP address via DNS (could be using a service like AWS Route 53).
5. **Cert-Manager**: Cert-Manager will manage your SSL/TLS certificates.
6. **Let’s Encrypt**: Free SSL certificate provider to automatically issue and renew certificates.

### **Step-by-Step Production Implementation**

---

### **Step 1: Deploy the NGINX Application**

#### **1.1. Create a Deployment for NGINX**

We'll first create an NGINX Deployment and expose it with a Service.

**nginx-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Apply the deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

#### **1.2. Expose the NGINX Deployment with a Service**

Expose the deployment internally using a **ClusterIP** service.

**nginx-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

Apply the service:

```bash
kubectl apply -f nginx-service.yaml
```

---

### **Step 2: Install NGINX Ingress Controller**

If you don’t already have an **NGINX Ingress Controller** running, install it as follows.

#### **2.1. Create the NGINX Ingress Controller Deployment**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

Wait for the NGINX Ingress Controller to be ready:

```bash
kubectl get pods -n ingress-nginx
```

You should see the **ingress-nginx-controller** pod running.

---

### **Step 3: Configure Ingress for Routing**

Now, we will configure an **Ingress resource** that will route external traffic to our NGINX application.

#### **3.1. Create the Ingress Resource**

**nginx-ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: nginx.example.com     # Replace with your actual domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

**Explanation:**
- **host**: Replace `nginx.example.com` with your real domain name.
- **service**: The service that routes traffic to your NGINX pods.

Apply the Ingress resource:

```bash
kubectl apply -f nginx-ingress.yaml
```

At this point, you should be able to access your application via HTTP at `http://nginx.example.com`. But we need to secure this with HTTPS.

---

### **Step 4: Install Cert-Manager for TLS Certificates**

Cert-Manager automatically manages certificates in Kubernetes.

#### **4.1. Install Cert-Manager**

To install Cert-Manager in your cluster, apply the following manifest:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

Verify Cert-Manager is running:

```bash
kubectl get pods --namespace cert-manager
```

Ensure that Cert-Manager components (`cert-manager`, `cert-manager-webhook`, and `cert-manager-cainjector`) are running.

---

### **Step 5: Configure Cert-Manager with Let’s Encrypt**

#### **5.1. Create a ClusterIssuer for Let’s Encrypt**

Cert-Manager requires an **Issuer** to know where to get the certificates. We’ll use Let’s Encrypt for this.

Create a YAML file for the **ClusterIssuer**:

**letsencrypt-clusterissuer.yaml**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com    # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply the **ClusterIssuer**:

```bash
kubectl apply -f letsencrypt-clusterissuer.yaml
```

This configuration tells Cert-Manager to use Let’s Encrypt to generate certificates using the HTTP-01 challenge with the NGINX Ingress controller.

---

### **Step 6: Update Ingress to Enable TLS**

We will now modify our Ingress resource to use Cert-Manager to secure the application with **TLS**.

#### **6.1. Modify the Ingress Resource**

**nginx-ingress-tls.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # Reference to Cert-Manager's ClusterIssuer
spec:
  tls:
  - hosts:
    - nginx.example.com      # Your domain
    secretName: nginx-tls    # Cert-Manager will create this secret
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

**Explanation:**
- **tls**: Specifies that TLS should be used for the host `nginx.example.com`. Cert-Manager will create a secret (`nginx-tls`) to store the certificate and private key.

Apply the updated Ingress:

```bash
kubectl apply -f nginx-ingress-tls.yaml
```

---

### **Step 7: Verify Certificate Issuance**

Cert-Manager will automatically request a certificate from Let’s Encrypt based on the Ingress configuration.

#### **7.1. Check Certificate Status**

You can monitor the certificate issuance by checking the `Certificate` resource created by Cert-Manager:

```bash
kubectl get certificates
```

Check its status using:

```bash
kubectl describe certificate nginx-tls
```

If successful, you'll see that the certificate has been issued.

#### **7.2. Verify the Secret**

Cert-Manager stores the certificate and private key in a Kubernetes **Secret**. Verify its creation:

```bash
kubectl get secret nginx-tls
```

---

### **Step 8: Test HTTPS Access**

Now, visit `https://nginx.example.com` in your browser. You should see:
- The NGINX application served over HTTPS.
- A valid Let's Encrypt certificate.

### **Step 9: Cert-Manager Automates Certificate Renewal**

Cert-Manager will automatically renew the certificate before it expires, so there’s no manual intervention required.

---

### **Why Move from ClusterIP → NodePort → LoadBalancer → Ingress?**

1. **ClusterIP**: It’s only accessible within the cluster. It’s useful for internal services but cannot handle external traffic.

2. **NodePort**: Allows external traffic but exposes the service on a specific port (range 30000-32767) on every node. It's not scalable and is limited in port availability.

3. **LoadBalancer**: Automatically creates an external load balancer (in cloud environments like AWS/GCP) to route traffic. However, it creates a load balancer for each service, which can be costly and difficult to manage at scale.

4. **Ingress**: Instead of provisioning a load balancer for every service, Ingress provides a single entry point to multiple services. It also provides additional functionality like path-based routing, SSL termination, and handling multiple domains. Ingress is much more scalable and manageable for complex production environments.

---

This completes the **end-to-end** production-level setup of an **Ingress** with TLS security using **Cert-Manager** and **Let’s Encrypt**. This architecture is highly scalable, cost-efficient, and secure for any production-grade Kubernetes deployment.
