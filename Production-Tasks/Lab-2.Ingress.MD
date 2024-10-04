Deploying applications with a Dockerized image from Docker Hub, setting up ingress routing, and configuring SSL termination. Each step will include detailed explanations for hands-on implementation. 

---

## **Step-by-Step Implementation Guide for Path-Based and Host-Based Ingress with SSL Termination**

### **Step 1: Set Up DNS in Route 53**

Before deploying the application, you need to configure your DNS for custom domains using **Amazon Route 53**. This will enable your custom domains to map to the Kubernetes cluster and route traffic properly.

#### 1.1 Create Hosted Zone in Route 53
- A **Hosted Zone** is a container for DNS records of a domain.
  
Steps to create a **Hosted Zone**:
1. Go to **Route 53** in the AWS Management Console.
2. Click on **Create Hosted Zone**.
3. In the **Domain Name** field, enter your domain name (e.g., `naveenops.live`).
4. Select **Public Hosted Zone** and click **Create**.
  
This creates a hosted zone for `naveenops.live`. Route 53 will provide **Name Server (NS) Records**. You’ll need to update your domain registrar (like GoDaddy) with these NS records to point the domain to AWS.

#### 1.2 Create DNS Records (A Records) for Subdomains

To route traffic for subdomains like `example.naveenops.live`, `sample-1.naveenops.live`, and `sample-2.naveenops.live`, create **A Records** pointing to your **Elastic Load Balancer (ELB)** or NGINX Ingress Controller IP.

After setting up the NGINX Ingress Controller (Step 3), you’ll retrieve the ELB DNS Name and use that to create the **A Records**.

##### Example A Records:
1. **A Record** for `example.naveenops.live`:
   - **Name**: `example.naveenops.live`
   - **Type**: A
   - **Value**: The **ELB DNS Name** created by the NGINX Ingress Controller.

2. **A Record** for `sample-1.naveenops.live`:
   - **Name**: `sample-1.naveenops.live`
   - **Type**: A
   - **Value**: **ELB DNS Name**

3. **A Record** for `sample-2.naveenops.live`:
   - **Name**: `sample-2.naveenops.live`
   - **Type**: A
   - **Value**: **ELB DNS Name**

- Go to **Route 53** → **Hosted Zones** → Select your domain → **Create Record**.
- Enter the subdomain name (`example.naveenops.live`, etc.) and map it to the **ELB DNS Name** or **IP** from your NGINX Ingress Controller.

---

## **Step 2: Download Dockerized Application from Docker Hub**

Before deploying the applications, let's pull the Docker images from Docker Hub.

### 2.1 Pull Docker Images from Docker Hub

To deploy an application on Kubernetes, you'll need a Docker image. Here, we'll use pre-existing images from **Docker Hub**.

### Example Docker Images:
- **Path-based applications**:
  - Image 1: `devopsprosamples/next-path-sample-1`
  - Image 2: `devopsprosamples/next-path-sample-2`
- **Host-based applications**:
  - Image 1: `devopsprosamples/next-sample-1`
  - Image 2: `devopsprosamples/next-sample-2`

Use the following command to pull the images:

```bash
docker pull devopsprosamples/next-path-sample-1
docker pull devopsprosamples/next-path-sample-2
docker pull devopsprosamples/next-sample-1
docker pull devopsprosamples/next-sample-2
```

Verify that the images are available locally by running:

```bash
docker images
```

Now, these images are ready to be deployed on Kubernetes.

---

## **Step 3: Install the NGINX Ingress Controller**

The **Ingress Controller** will manage external access to the services running inside the Kubernetes cluster. We’ll use the **NGINX Ingress Controller** for this setup.

### 3.1 Install NGINX Ingress Controller
Deploy the official NGINX Ingress Controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### 3.2 Verify Installation
Once the installation is complete, check that the ingress controller is running and a **LoadBalancer** service has been created (in AWS, this will automatically provision an ELB).

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

The output of `kubectl get svc -n ingress-nginx` should show a **LoadBalancer** with an external IP (or ELB DNS name) that you'll use for the Route 53 **A Records**.

---

## **Step 4: Deploy Applications on Kubernetes**

Now that the images have been pulled and the ingress controller is set up, we can deploy the applications on Kubernetes.

### 4.1 Create Kubernetes Deployments
We’ll create deployments for each of the applications using the images pulled from Docker Hub.

```bash
kubectl create deploy sample-1 --image=devopsprosamples/next-path-sample-1
kubectl create deploy sample-2 --image=devopsprosamples/next-path-sample-2
kubectl create deploy sample-3 --image=devopsprosamples/next-sample-1
kubectl create deploy sample-4 --image=devopsprosamples/next-sample-2
```

- These commands create **Deployments** for each application, which in turn will manage the pods for those applications.

### 4.2 Expose Deployments as Services
Expose each deployment as a **ClusterIP** service, which allows them to be accessible within the cluster at a specific port.

```bash
kubectl expose deploy sample-1 --type=ClusterIP --port=3000
kubectl expose deploy sample-2 --type=ClusterIP --port=3000
kubectl expose deploy sample-3 --type=ClusterIP --port=3000
kubectl expose deploy sample-4 --type=ClusterIP --port=3000
```

Each service will now listen on **port 3000** internally.

### 4.3 Verify Services
Run the following command to ensure that the services have been created successfully:

```bash
kubectl get svc
```

You should see four services listed, each listening on port 3000.

---

## **Step 5: Configure Ingress for Path-Based and Host-Based Routing**

Now, let’s configure the **Ingress** to manage routing based on **paths** and **hosts**.

### 5.1 Create Ingress YAML
We will create an Ingress resource that defines how traffic should be routed based on the domain and path.

#### Example `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /  # Rewrite the incoming path to the root path of the service.
    kubernetes.io/ingress.class: "nginx"           # Use the NGINX ingress controller for this ingress.
spec:
  rules:
  # Path-based routing for example.naveenops.live
  - host: "example.naveenops.live"                 # Host for the domain.
    http:
      paths:
      - pathType: Prefix
        path: /sample-1                            # Route /sample-1 to the sample-1 service.
        backend:
          service:
            name: sample-1
            port:
              number: 3000
      - pathType: Prefix
        path: /sample-2                            # Route /sample-2 to the sample-2 service.
        backend:
          service:
            name: sample-2
            port:
              number: 3000

  # Host-based routing for sample-1.naveenops.live and sample-2.naveenops.live
  - host: "sample-1.naveenops.live"                # Host-based routing for sample-1.naveenops.live.
    http:
      paths:
      - pathType: Prefix
        path: "/"                                  # Route all traffic to sample-3 service.
        backend:
          service:
            name: sample-3
            port:
              number: 3000

  - host: "sample-2.naveenops.live"                # Host-based routing for sample-2.naveenops.live.
    http:
      paths:
      - pathType: Prefix
        path: "/"                                  # Route all traffic to sample-4 service.
        backend:
          service:
            name: sample-4
            port:
              number: 3000
```

### Explanation:
- **Path-based Routing**:
  - Requests to `example.naveenops.live/sample-1` are routed to **sample-1**.
  - Requests to `example.naveenops.live/sample-2` are routed to **sample-2**.
  
- **Host-based Routing**:
  - Requests to `sample-1.naveenops.live` are routed to **sample-3**.
  - Requests to `sample-2.nave

enops.live` are routed to **sample-4**.

### 5.2 Apply the Ingress Configuration

```bash
kubectl apply -f ingress.yaml
```

Check the status of the ingress:

```bash
kubectl get ing
kubectl describe ingress example-ingress
```

---

## **Step 6: Configure SSL Termination with Cert-Manager and Let’s Encrypt**

To secure the traffic with SSL, we will use **Cert-Manager** to automate the SSL certificate issuance process with **Let’s Encrypt**.

### 6.1 Install Cert-Manager

Install **Cert-Manager** to manage SSL certificates:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

### 6.2 Verify Cert-Manager Installation

Check if Cert-Manager is running:

```bash
kubectl get pods -n cert-manager
kubectl get svc -n cert-manager
```

### 6.3 Create Let’s Encrypt ClusterIssuer

The **ClusterIssuer** is responsible for communicating with Let’s Encrypt to issue certificates.

Create `letsencrypt-clusterissuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory  # Let’s Encrypt production server
    email: your-email@example.com                           # Email for notifications
    privateKeySecretRef:
      name: letsencrypt-prod                                # Secret for the ACME account private key
    solvers:
    - http01:
        ingress:
          class: nginx                                      # Use the NGINX ingress controller
```

Apply the **ClusterIssuer**:

```bash
kubectl apply -f letsencrypt-clusterissuer.yaml
```

### 6.4 Update Ingress to Enable SSL

Update your **ingress.yaml** to include SSL configuration:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"      # Enable SSL with Let's Encrypt
spec:
  tls:
  - hosts:
    - example.naveenops.live                                # SSL for this host
    - sample-1.naveenops.live
    - sample-2.naveenops.live
    secretName: tls-secret                                  # Secret to store SSL certificates
  rules:
    # Path and host-based rules as defined earlier.
```

Apply the updated Ingress:

```bash
kubectl apply -f ingress.yaml
```

### 6.5 Verify SSL Certificates

Once the certificates are issued by Cert-Manager, you can verify the status:

```bash
kubectl get certificates
kubectl get secret
```

Access your services securely via **HTTPS**:

- **https://example.naveenops.live**
- **https://sample-1.naveenops.live**
- **https://sample-2.naveenops.live**

---

## **Summary**

1. **DNS Configuration**: Set up Route 53 for custom domains.
2. **Dockerized Application**: Pulled Docker images from Docker Hub and deployed the applications.
3. **Ingress Controller**: Installed and configured the NGINX Ingress Controller.
4. **Application Deployment**: Deployed applications and exposed them as services.
5. **Ingress Configuration**: Configured path-based and host-based routing using Kubernetes Ingress.
6. **SSL Termination**: Automated SSL certificate issuance using Cert-Manager and Let’s Encrypt.

This setup ensures your applications are accessible via custom domains with secure HTTPS routing.
