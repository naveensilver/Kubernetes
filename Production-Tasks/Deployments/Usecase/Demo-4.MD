Let's implement an end-to-end example of A/B testing using Kubernetes. In this scenario, we will deploy two versions of a web application (Version A and Version B) and use a Kubernetes Service to route traffic between them. This setup allows us to evaluate which version performs better based on user interactions.

### Scenario Overview

We will:
1. Create two versions of a simple web application.
2. Containerize each version using Docker.
3. Deploy both versions to a Kubernetes cluster using separate Deployments.
4. Set up a Service to route traffic between the two versions.
5. Demonstrate how to access each version.

### Step 1: Create the Web Applications

#### 1. Set Up the Project Directory
```bash
mkdir ab-testing
cd ab-testing
```

#### 2. Create Version A of the Application

1. **Create `app_a.py` for Version A**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def hello():
       return jsonify({"version": "A", "message": "Welcome to Version A!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create `requirements.txt` for Version A**:
   ```
   Flask==2.1.1
   ```

3. **Create a Dockerfile for Version A**:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app_a.py .

   EXPOSE 5000
   CMD ["python", "app_a.py"]
   ```

#### 3. Create Version B of the Application

1. **Create `app_b.py` for Version B**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def hello():
       return jsonify({"version": "B", "message": "Welcome to Version B!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create `requirements.txt` for Version B**:
   ```
   Flask==2.1.1
   ```

3. **Create a Dockerfile for Version B**:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app_b.py .

   EXPOSE 5000
   CMD ["python", "app_b.py"]
   ```

### Step 2: Build and Push Docker Images

#### 1. Build and Push Each Version

1. **For Version A**:
   ```bash
   cd version_a
   docker build -t <your-dockerhub-username>/ab-testing-a:latest .
   docker push <your-dockerhub-username>/ab-testing-a:latest
   cd ..
   ```

2. **For Version B**:
   ```bash
   cd version_b
   docker build -t <your-dockerhub-username>/ab-testing-b:latest .
   docker push <your-dockerhub-username>/ab-testing-b:latest
   cd ..
   ```

### Step 3: Deploy Versions A and B to Kubernetes

#### 1. Create Deployment YAML Files

1. **For Version A (`deployment_a.yaml`)**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ab-testing-a
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: ab-testing-a
     template:
       metadata:
         labels:
           app: ab-testing-a
       spec:
         containers:
         - name: ab-testing-a
           image: <your-dockerhub-username>/ab-testing-a:latest
           ports:
           - containerPort: 5000
   ```

2. **For Version B (`deployment_b.yaml`)**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ab-testing-b
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: ab-testing-b
     template:
       metadata:
         labels:
           app: ab-testing-b
       spec:
         containers:
         - name: ab-testing-b
           image: <your-dockerhub-username>/ab-testing-b:latest
           ports:
           - containerPort: 5000
   ```

#### 2. Deploy Each Version

1. **Apply Version A**:
   ```bash
   kubectl apply -f deployment_a.yaml
   ```

2. **Apply Version B**:
   ```bash
   kubectl apply -f deployment_b.yaml
   ```

### Step 4: Expose the Applications

1. **Create a Service YAML File** (`ab-testing-service.yaml`):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: ab-testing
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: ab-testing-a
   ```

2. **Apply the Service**:
   ```bash
   kubectl apply -f ab-testing-service.yaml
   ```

### Step 5: Implement Traffic Splitting

To split traffic between Version A and Version B, we can update the Service to use an ExternalTrafficPolicy and configure it to route to both deployments. Alternatively, we can create two Services for each version and manually control traffic.

#### 1. Update the Service

1. **Update `ab-testing-service.yaml`** to route to both versions:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: ab-testing
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: ab-testing
   ```

#### 2. Use Ingress for Traffic Splitting (Optional)

If you want to use more advanced traffic routing, consider using an Ingress Controller with rules that can route traffic based on headers or paths. Here's a simple example of how to set up Ingress.

1. **Create an Ingress Resource (`ab-testing-ingress.yaml`)**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: ab-testing-ingress
   spec:
     rules:
     - host: ab-testing.example.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: ab-testing
               port:
                 number: 80
   ```

### Step 6: Test the A/B Testing Setup

1. **Access the Load Balancer**:
   Use `kubectl get services` to find the external IP of the `ab-testing` service and access it.

2. **Check Responses**:
   Visit the service endpoint multiple times to see responses from both Version A and Version B.

### Step 7: Clean Up

1. **Delete the Services and Deployments**:
   ```bash
   kubectl delete -f deployment_a.yaml
   kubectl delete -f deployment_b.yaml
   kubectl delete -f ab-testing-service.yaml
   kubectl delete -f ab-testing-ingress.yaml
   ```

### Conclusion

In this end-to-end implementation, we:
- Created two versions of a web application.
- Containerized each version using Docker.
- Deployed both versions to Kubernetes using separate Deployments.
- Set up a Service (and optionally Ingress) to manage traffic and enable A/B testing.

This allows for evaluating which version performs better in a production-like environment. If you have further questions or need assistance, feel free to ask!
