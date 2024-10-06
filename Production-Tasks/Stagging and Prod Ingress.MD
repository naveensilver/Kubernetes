### Comprehensive Guide to Setting Up SSL/TLS Management with Cert-Manager and NGINX Ingress in Kubernetes Environments

Let’s go through the entire process of setting up Cert-Manager and NGINX Ingress Controller in both staging and production environments in a Kubernetes cluster. We will deploy a sample application with SSL/TLS certificates managed by Cert-Manager.

### Environment Setup

1. **Assumptions**:
   - You have access to a Kubernetes cluster.
   - You have `kubectl` configured and working.
   - You have a domain name pointing to your Ingress Controller's external IP.

### Step-by-Step Implementation

#### Part 1: Install NGINX Ingress Controller

##### 1. Create Namespaces

Create separate namespaces for staging and production:

```bash
kubectl create namespace ingress-nginx-staging
kubectl create namespace ingress-nginx-production
```

##### 2. Deploy NGINX Ingress Controller

Use the following YAML manifest to deploy the NGINX Ingress Controller in both namespaces.

```yaml
# nginx-ingress-controller.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx-staging  # Change to ingress-nginx-production for production
  labels:
    app: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress-controller
  template:
    metadata:
      labels:
        app: nginx-ingress-controller
    spec:
      containers:
      - name: nginx-ingress-controller
        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:latest
        args:
          - /nginx-ingress-controller
          - --configmap=$(POD_NAMESPACE)/nginx-configuration
          - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
          - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
          - --watch-namespace=""
        ports:
          - name: http
            containerPort: 80
          - name: https
            containerPort: 443
        env:
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
```

**Apply the Deployment**:
```bash
kubectl apply -f nginx-ingress-controller.yaml
```

##### 3. Create Service for NGINX Ingress Controller

Use the following YAML manifest to create a Service in both namespaces.

```yaml
# nginx-ingress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-service
  namespace: ingress-nginx-staging  # Change to ingress-nginx-production for production
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
    - port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app: nginx-ingress-controller
```

**Apply the Service**:
```bash
kubectl apply -f nginx-ingress-service.yaml
```

---

#### Part 2: Install Cert-Manager

##### 1. Create Cert-Manager Namespace

Create the Cert-Manager namespace:

```bash
kubectl create namespace cert-manager
```

##### 2. Deploy Cert-Manager Custom Resource Definitions (CRDs)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
```

##### 3. Deploy Cert-Manager Components

Use the following YAML manifest to deploy Cert-Manager:

```yaml
# cert-manager.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cert-manager
  template:
    metadata:
      labels:
        app: cert-manager
    spec:
      containers:
      - name: cert-manager
        image: quay.io/jetstack/cert-manager-controller:v1.11.0
        args:
          - --v=2
          - --cluster-resource-namespace=$(POD_NAMESPACE)
        env:
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
```

**Apply Cert-Manager**:
```bash
kubectl apply -f cert-manager.yaml
```

---

#### Part 3: Configure Issuers

##### 1. Create ClusterIssuer for Staging and Production

**Staging Issuer**:

```yaml
# letsencrypt-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging.api.letsencrypt.org/directory
    email: your-email@example.com  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Production Issuer**:

```yaml
# letsencrypt-production.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-production-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Apply the Issuers**:
```bash
kubectl apply -f letsencrypt-staging.yaml
kubectl apply -f letsencrypt-production.yaml
```

---

#### Part 4: Deploy Sample Application

##### 1. Create NGINX Deployment

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 1
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
        image: nginx
        ports:
        - containerPort: 80
```

**Apply the Deployment**:
```bash
kubectl apply -f nginx-deployment.yaml
```

##### 2. Expose Deployment as a Service

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
```

**Apply the Service**:
```bash
kubectl apply -f nginx-service.yaml
```

---

#### Part 5: Create Certificates and Ingress Resource

##### 1. Create Certificates

**Staging Certificate**:

```yaml
# nginx-staging-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nginx-cert-staging
  namespace: default
spec:
  secretName: nginx-cert-staging-secret
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: staging.example.com  # Replace with your staging domain
  dnsNames:
    - staging.example.com            # Replace with your staging domain
```

**Production Certificate**:

```yaml
# nginx-production-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nginx-cert-production
  namespace: default
spec:
  secretName: nginx-cert-production-secret
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: example.com           # Replace with your production domain
  dnsNames:
    - example.com                    # Replace with your production domain
```

**Apply the Certificates**:
```bash
kubectl apply -f nginx-staging-certificate.yaml
kubectl apply -f nginx-production-certificate.yaml
```

##### 2. Create Ingress Resources

**Staging Ingress**:

```yaml
# nginx-staging-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-staging
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - staging.example.com            # Replace with your staging domain
    secretName: nginx-cert-staging-secret
  rules:
  - host: staging.example.com         # Replace with your staging domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

**Production Ingress**:

```yaml
# nginx-production-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-production
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - example.com                     # Replace with your production domain
    secretName: nginx-cert-production-secret
  rules:
  - host: example.com                  # Replace with your production domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

**Apply the Ingress Resources**:
```bash
kubectl apply -f nginx-staging-ingress.yaml
kubectl apply -f nginx-production-ingress.yaml
```

---

### Step

 6: Verify the Setup

1. **Check Certificate Status**:
   ```bash
   kubectl describe certificate nginx-cert-staging -n default
   kubectl describe certificate nginx-cert-production -n default
   ```

2. **Access Applications**:
   - For Staging: Access `https://staging.example.com`
   - For Production: Access `https://example.com`

3. **Check Ingress**:
   ```bash
   kubectl get ingress -n default
   ```

### Summary

- **NGINX Ingress Controller**: Installed in both staging and production environments.
- **Cert-Manager**: Installed to handle SSL/TLS certificates automatically.
- **Issuers**: Created for staging and production environments.
- **Sample Application**: Deployed using NGINX, with services and ingress configured to use certificates.

This comprehensive guide provides a clear step-by-step process for implementing SSL/TLS management with Cert-Manager and NGINX Ingress Controller in both staging and production environments. If you have further questions or need additional assistance, feel free to ask!
