Let's implement an end-to-end example of a Continuous Integration/Continuous Deployment (CI/CD) pipeline for a Kubernetes application. We’ll set up a simple CI/CD process that automates the deployment of application updates whenever new code is pushed to a GitHub repository. This example will involve:

1. **Creating a simple web application.**
2. **Setting up a GitHub repository.**
3. **Configuring a CI/CD tool (GitHub Actions) to build and deploy to Kubernetes.**

### Step 1: Create the Web Application

#### 1. Set Up the Project Directory
```bash
mkdir ci-cd-example
cd ci-cd-example
```

#### 2. Create the Web Application

1. **Create `app.py`**:
   ```python
   from flask import Flask, jsonify

   app = Flask(__name__)

   @app.route('/')
   def hello():
       return jsonify({"message": "Hello, CI/CD!"})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

2. **Create `requirements.txt`**:
   ```
   Flask==2.1.1
   ```

3. **Create a Dockerfile**:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app.py .

   EXPOSE 5000
   CMD ["python", "app.py"]
   ```

### Step 2: Create a GitHub Repository

1. **Create a new repository** on GitHub named `ci-cd-example`.

2. **Push your local code to the repository**:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/<your-username>/ci-cd-example.git
   git push -u origin master
   ```

### Step 3: Set Up GitHub Actions for CI/CD

1. **Create the GitHub Actions Workflow**:
   Create a directory called `.github/workflows` in your project root. Inside that directory, create a file named `ci-cd-pipeline.yml`:

   ```yaml
   name: CI/CD Pipeline

   on:
     push:
       branches:
         - master

   jobs:
     build:
       runs-on: ubuntu-latest

       steps:
         - name: Checkout code
           uses: actions/checkout@v2

         - name: Set up Docker Buildx
           uses: docker/setup-buildx-action@v1

         - name: Login to Docker Hub
           uses: docker/login-action@v1
           with:
             username: ${{ secrets.DOCKER_USERNAME }}
             password: ${{ secrets.DOCKER_PASSWORD }}

         - name: Build and push Docker image
           uses: docker/build-push-action@v2
           with:
             context: .
             push: true
             tags: <your-dockerhub-username>/ci-cd-example:latest

     deploy:
       runs-on: ubuntu-latest
       needs: build
       steps:
         - name: Set up kubectl
           uses: azure/setup-kubectl@v1
           with:
             version: 'latest'

         - name: Configure kubectl
           run: |
             echo "${{ secrets.KUBE_CONFIG }}" | base64 --decode > $HOME/.kube/config

         - name: Deploy to Kubernetes
           run: |
             kubectl set image deployment/ci-cd-example ci-cd-example=<your-dockerhub-username>/ci-cd-example:latest --record
   ```

### Step 4: Set Up Secrets in GitHub

1. **Docker Hub Credentials**:
   - Go to your GitHub repository > Settings > Secrets and variables > Actions.
   - Add `DOCKER_USERNAME` and `DOCKER_PASSWORD` as secrets.

2. **Kubernetes Configuration**:
   - Generate a base64 encoded Kubeconfig:
     ```bash
     cat ~/.kube/config | base64
     ```
   - Add the output as a secret named `KUBE_CONFIG` in your GitHub repository secrets.

### Step 5: Create Deployment Configuration

1. **Create a Deployment YAML File** (`deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ci-cd-example
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: ci-cd-example
     template:
       metadata:
         labels:
           app: ci-cd-example
       spec:
         containers:
         - name: ci-cd-example
           image: <your-dockerhub-username>/ci-cd-example:latest
           ports:
           - containerPort: 5000
   ```

2. **Apply the Deployment**:
   ```bash
   kubectl apply -f deployment.yaml
   ```

### Step 6: Test the CI/CD Pipeline

1. **Make a Change**:
   Edit `app.py` to change the message:
   ```python
   @app.route('/')
   def hello():
       return jsonify({"message": "Hello, CI/CD with GitHub Actions!"})
   ```

2. **Commit and Push**:
   ```bash
   git add app.py
   git commit -m "Update message"
   git push
   ```

3. **Check GitHub Actions**:
   Go to your GitHub repository and check the Actions tab to see the pipeline running.

### Step 7: Verify the Deployment

1. **Get the Pods**:
   ```bash
   kubectl get pods
   ```

2. **Test the Application**:
   Use `kubectl port-forward` to access the application:
   ```bash
   kubectl port-forward deployment/ci-cd-example 8080:5000
   ```
   Visit `http://localhost:8080` to see the updated message.

### Step 8: Clean Up

1. **Delete the Deployment**:
   ```bash
   kubectl delete -f deployment.yaml
   ```

### Conclusion

In this end-to-end implementation, we:
- Created a simple web application using Flask.
- Set up a GitHub repository and integrated GitHub Actions for CI/CD.
- Automated the process of building a Docker image and deploying it to Kubernetes when code changes are pushed.

This CI/CD pipeline enables continuous updates and ensures that your application can be deployed rapidly and reliably. If you have any questions or need further assistance, feel free to ask!
