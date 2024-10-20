Let’s implement an end-to-end example of deploying a microservices architecture for an e-commerce platform using Kubernetes. We will create three separate microservices: **User Authentication**, **Product Catalog**, and **Payment Processing**. Each service will be containerized, deployed independently, and managed through Kubernetes Deployments.

### Scenario Overview

We'll:
1. Create three simple microservices using Flask.
2. Containerize each service using Docker.
3. Deploy each service to a Kubernetes cluster using separate Deployments.
4. Create a Service for each microservice to expose them.
5. Demonstrate how to manage and scale each service independently.

### Step 1: Create Microservices

#### 1. Set Up the Project Directory
```bash
mkdir ecommerce-platform
cd ecommerce-platform
```

#### 2. Create Microservice Directories
```bash
mkdir auth-service product-service payment-service
```

#### 3. User Authentication Service

1. **Create `app.py` for User Authentication** in `auth-service/`:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/auth')
   def authenticate():
       return jsonify({"message": "User authenticated!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create `requirements.txt` for User Authentication**:
   ```
   Flask==2.1.1
   ```

3. **Create a Dockerfile** in `auth-service/`:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app.py .

   EXPOSE 5000
   CMD ["python", "app.py"]
   ```

#### 4. Product Catalog Service

1. **Create `app.py` for Product Catalog** in `product-service/`:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/products')
   def get_products():
       return jsonify({"products": ["Product 1", "Product 2", "Product 3"]})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create `requirements.txt` for Product Catalog**:
   ```
   Flask==2.1.1
   ```

3. **Create a Dockerfile** in `product-service/`:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app.py .

   EXPOSE 5000
   CMD ["python", "app.py"]
   ```

#### 5. Payment Processing Service

1. **Create `app.py` for Payment Processing** in `payment-service/`:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/pay')
   def pay():
       return jsonify({"message": "Payment processed!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create `requirements.txt` for Payment Processing**:
   ```
   Flask==2.1.1
   ```

3. **Create a Dockerfile** in `payment-service/`:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app.py .

   EXPOSE 5000
   CMD ["python", "app.py"]
   ```

### Step 2: Build and Push Docker Images

#### 1. Build and Push Each Service

1. **For Auth Service**:
   ```bash
   cd auth-service
   docker build -t <your-dockerhub-username>/auth-service:latest .
   docker push <your-dockerhub-username>/auth-service:latest
   cd ..
   ```

2. **For Product Service**:
   ```bash
   cd product-service
   docker build -t <your-dockerhub-username>/product-service:latest .
   docker push <your-dockerhub-username>/product-service:latest
   cd ..
   ```

3. **For Payment Service**:
   ```bash
   cd payment-service
   docker build -t <your-dockerhub-username>/payment-service:latest .
   docker push <your-dockerhub-username>/payment-service:latest
   cd ..
   ```

### Step 3: Deploy Microservices to Kubernetes

#### 1. Create Deployment YAML Files

1. **For User Authentication Service** (`auth-deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: auth-service
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: auth-service
     template:
       metadata:
         labels:
           app: auth-service
       spec:
         containers:
         - name: auth-service
           image: <your-dockerhub-username>/auth-service:latest
           ports:
           - containerPort: 5000
   ```

2. **For Product Catalog Service** (`product-deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: product-service
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: product-service
     template:
       metadata:
         labels:
           app: product-service
       spec:
         containers:
         - name: product-service
           image: <your-dockerhub-username>/product-service:latest
           ports:
           - containerPort: 5000
   ```

3. **For Payment Processing Service** (`payment-deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: payment-service
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: payment-service
     template:
       metadata:
         labels:
           app: payment-service
       spec:
         containers:
         - name: payment-service
           image: <your-dockerhub-username>/payment-service:latest
           ports:
           - containerPort: 5000
   ```

#### 2. Deploy Each Service

1. **Apply the Auth Service**:
   ```bash
   kubectl apply -f auth-deployment.yaml
   ```

2. **Apply the Product Service**:
   ```bash
   kubectl apply -f product-deployment.yaml
   ```

3. **Apply the Payment Service**:
   ```bash
   kubectl apply -f payment-deployment.yaml
   ```

#### 3. Verify Deployments
Check the status of the Deployments:
```bash
kubectl get deployments
kubectl get pods
```

### Step 4: Expose Each Microservice

#### 1. Create Service YAML Files

1. **For User Authentication Service** (`auth-service.yaml`):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: auth-service
   spec:
     type: ClusterIP
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: auth-service
   ```

2. **For Product Catalog Service** (`product-service.yaml`):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: product-service
   spec:
     type: ClusterIP
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: product-service
   ```

3. **For Payment Processing Service** (`payment-service.yaml`):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: payment-service
   spec:
     type: ClusterIP
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: payment-service
   ```

#### 2. Apply Each Service

1. **Apply the Auth Service**:
   ```bash
   kubectl apply -f auth-service.yaml
   ```

2. **Apply the Product Service**:
   ```bash
   kubectl apply -f product-service.yaml
   ```

3. **Apply the Payment Service**:
   ```bash
   kubectl apply -f payment-service.yaml
   ```

### Step 5: Test the Microservices

#### 1. Access the Services
You can access the services from within the Kubernetes cluster. For testing, you can use `kubectl port-forward` to expose the services locally:

1. **For Auth Service**:
   ```bash
   kubectl port-forward service/auth-service 8080:80
   ```
   Visit `http://localhost:8080/auth` to see the response.

2. **For Product Service**:
   ```bash
   kubectl port-forward service/product-service 8081:80
   ```
   Visit `http://localhost:8081/products` to see the response.

3. **For Payment Service**:
   ```bash
   kubectl port-forward service/payment-service 8082:80
   ```
   Visit `http://localhost:8082/pay` to see the response.

### Step 6: Scale Services Independently

#### 1. Scale the User Authentication Service
```bash
kubectl scale deployment auth

-service --replicas=3
```

#### 2. Scale the Product Catalog Service
```bash
kubectl scale deployment product-service --replicas=4
```

#### 3. Scale the Payment Processing Service
```bash
kubectl scale deployment payment-service --replicas=2
```

### Step 7: Clean Up

1. **Delete the Services**:
   ```bash
   kubectl delete -f auth-service.yaml
   kubectl delete -f product-service.yaml
   kubectl delete -f payment-service.yaml
   ```

2. **Delete the Deployments**:
   ```bash
   kubectl delete -f auth-deployment.yaml
   kubectl delete -f product-deployment.yaml
   kubectl delete -f payment-deployment.yaml
   ```

### Conclusion

In this hands-on implementation, we:
- Created three independent microservices for an e-commerce platform.
- Containerized each service with Docker.
- Deployed each service in Kubernetes using Deployments and Services.
- Scaled the services independently based on demand.

This microservices architecture allows for efficient management, scaling, and updating of individual components without affecting the entire system. If you have any questions or need further assistance, feel free to ask!
