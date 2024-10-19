Let’s implement an end-to-end example of using probes in a serverless framework running on Kubernetes. In this scenario, we will deploy a simple serverless function using **Kubeless** (a popular serverless framework for Kubernetes) and configure a readiness probe to ensure that the function is fully initialized before it starts accepting traffic.

### Scenario Overview

We will:
1. Deploy a serverless function using Kubeless.
2. Configure a readiness probe to check if the function is ready to handle requests.
3. Demonstrate how the readiness probe affects traffic routing.

### Prerequisites

1. **Kubernetes Cluster**: You should have access to a Kubernetes cluster.
2. **Kubeless Installed**: Ensure that Kubeless is installed on your cluster. Follow [Kubeless installation instructions](https://kubeless.io/docs/quick-start/) if it is not installed.

### Step 1: Create the Serverless Function

1. **Create a Simple Function**: Create a file named `hello.py`.

   ```python
   def hello(event, context):
       return {
           'statusCode': 200,
           'body': 'Hello, World!'
       }
   ```

2. **Deploy the Function with a Readiness Probe**: Create a file named `kubeless-function.yaml`.

   ```yaml
   apiVersion: kubeless.io/v1
   kind: Function
   metadata:
     name: hello-function
   spec:
     handler: hello.hello
     runtime: python:3.8
     function: |
       def hello(event, context):
           return {
               'statusCode': 200,
               'body': 'Hello, World!'
           }
     requirements: |
       # requirements.txt content goes here
   ```

3. **Add Readiness Probe Configuration**: Modify the deployment to include a readiness probe. Note that Kubeless manages the deployments, so you'll need to update the deployment directly. After creating the function, find the generated deployment and add the readiness probe.

   ```bash
   kubectl get deployment -l function=hello-function
   ```

   Once you find the deployment, use `kubectl edit deployment <deployment-name>` to add the readiness probe:

   ```yaml
   readinessProbe:
     httpGet:
       path: /
       port: 8080
     initialDelaySeconds: 5
     periodSeconds: 5
   ```

### Step 2: Deploy the Function

1. **Deploy the Function**:
   ```bash
   kubectl apply -f kubeless-function.yaml
   ```

2. **Check the Function Status**:
   ```bash
   kubectl get functions
   ```

   You should see your function listed.

### Step 3: Verify the Deployment

1. **Check the Pods**:
   ```bash
   kubectl get pods
   ```

   You should see the Pods created for your function.

### Step 4: Test the Readiness Probe

1. **Get the Service**:
   Each function is exposed via a service. Retrieve the service details:
   ```bash
   kubectl get services
   ```

   Find the service corresponding to your function.

2. **Test Access**:
   Use `kubectl port-forward` to access your function locally:
   ```bash
   kubectl port-forward svc/hello-function 8080:8080
   ```

3. **Check Function Health**:
   While the function is starting, you can check if it responds. If the readiness probe is not satisfied, the service should not respond:
   ```bash
   curl http://localhost:8080/
   ```

   Initially, you may receive an error if it is not ready.

4. **Check Logs**:
   Monitor the logs to see when the function is ready:
   ```bash
   kubectl logs <hello-function-pod-name>
   ```

### Step 5: Update the Function

1. **Update the Function Code**: Modify `hello.py` to include a delay to simulate initialization:
   ```python
   import time

   def hello(event, context):
       time.sleep(10)  # Simulate a delay
       return {
           'statusCode': 200,
           'body': 'Hello, World!'
       }
   ```

2. **Redeploy the Function**:
   ```bash
   kubectl apply -f kubeless-function.yaml
   ```

3. **Monitor Deployment**: Check the Pods to observe their status:
   ```bash
   kubectl get pods -w
   ```

### Step 6: Clean Up

1. **Delete the Function and Related Resources**:
   ```bash
   kubectl delete -f kubeless-function.yaml
   ```

### Explanation of Key Components

- **Readiness Probe**: The readiness probe checks the function’s health by hitting the specified path. If the probe fails, the service will not route traffic to that Pod, ensuring that only ready instances serve requests.

- **Serverless Function**: Kubeless allows deploying functions that can respond to events. The readiness probe helps manage the lifecycle of these functions, especially during deployment and scaling events.

### Conclusion

In this implementation, we:
- Deployed a serverless function using Kubeless.
- Configured a readiness probe to ensure the function is ready to handle requests.
- Demonstrated how the readiness probe affects traffic routing during deployment and initialization.

This example showcases how to integrate readiness probes into serverless frameworks to maintain application availability and performance.
