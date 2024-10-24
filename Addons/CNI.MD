# Read This, Before diving into Network Policies in Kubernetes

CNI, or Container Network Interface, is a specification and framework used in Kubernetes (K8s) to manage network connectivity for containers. It defines a standard way for different network providers to configure and manage the networking of containerized applications. Here’s a quick overview:

### Key Concepts

1. **Plugins**: CNI allows various network plugins to be used, such as Calico, Flannel, Weave Net, and Cilium, each providing different networking capabilities and policies.

2. **Network Configuration**: When a pod is created, Kubernetes calls the appropriate CNI plugin to set up networking for that pod. This includes assigning an IP address and setting up routes.

3. **Network Policies**: Some CNI plugins support network policies, allowing you to control traffic flow between pods, enhancing security.

4. **Multitenancy**: CNI can help manage networking in multi-tenant environments, allowing for isolation and control over network resources.

### How It Works

1. **Pod Lifecycle**: When a pod is created, Kubernetes triggers the CNI plugin to create a network interface for the pod.
  
2. **IP Address Allocation**: The CNI plugin assigns an IP address to the pod from a defined subnet.

3. **Network Configuration**: The plugin configures routes and any necessary firewall rules.

4. **Pod Deletion**: When the pod is deleted, Kubernetes calls the CNI plugin again to clean up the network resources.

### Benefits

- **Flexibility**: Users can choose from various networking solutions based on their needs.
- **Scalability**: CNI supports dynamic scaling of networks as more pods are created.
- **Interoperability**: Provides a standardized interface for different network providers to work with Kubernetes.

CNI is essential for ensuring that pods can communicate effectively within a Kubernetes cluster and with external networks.

---------------------

### Check CNI Plugins: 
```
kubectl get pods -n kube-system
```
Look for a pod related to your CNI. (eg., cilium or calico)

>> Note: By default, a Kubernetes cluster does not come with a CNI plugin installed. To enable pod networking, you'll need to manually install a CNI plugin after setting up your cluster.

### Step-by-Step Guide to Install Calico with kops

**To install Calico in a Kubernetes cluster managed by kops, you can follow these steps:**

1. **Ensure kops is Installed**: Make sure you have kops installed and configured to manage your Kubernetes cluster. You can check this by running:
   ```bash
   kops version
   ```

2. **Create or Update Your Cluster**:
   If you’re creating a new cluster, you can specify Calico as your network provider during the creation process. If you already have a cluster, you can modify it to use Calico.

   To create a new cluster with Calico:
   ```bash
   kops create cluster \
       --name=mycluster.k8s.local \
       --cloud=aws \
       --zones=us-east-1a \
       --networking=calico \
       --yes
   ```

   For an existing cluster, update the cluster configuration:
   ```bash
   kops edit cluster --name=mycluster.k8s.local
   ```
   Then, change the networking field to:
   ```yaml
   networking:
     calico: {}
   ```

3. **Apply the Changes**:
   After editing the cluster configuration, apply the changes:
   ```bash
   kops update cluster --name=mycluster.k8s.local --yes
   ```

4. **Verify Calico Installation**:
   Once the update is complete, verify that the Calico pods are running:
   ```bash
   kubectl get pods -n kube-system
   ```
   Look for pods with names like `calico-node` and `calico-kube-controllers`.

5. **Check Network Policies (Optional)**:
   If you want to experiment with Calico's network policy features, you can create a sample network policy to restrict traffic between pods. For example:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: deny-all
     namespace: default
   spec:
     podSelector: {}
     policyTypes:
       - Ingress
   ```

   Apply it using:
   ```bash
   kubectl apply -f <your-policy-file>.yaml
   ```

### Additional Notes

- **Documentation**: Always refer to the official [kops documentation](https://kops.sigs.k8s.io/) and the [Calico documentation](https://docs.projectcalico.org/) for the most up-to-date instructions and best practices.
- **Network Configuration**: Ensure that your VPC and subnets are correctly set up to support Calico networking.

By following these steps, you should be able to successfully install and configure Calico in your kops-managed Kubernetes cluster!

---------------

CNI plugins in Kubernetes are often deployed as DaemonSets. Here's how this works and why it's done:

### What is a DaemonSet?

A DaemonSet is a Kubernetes resource that ensures that a copy of a specific pod runs on all (or a subset of) nodes in a cluster. This is particularly useful for deploying network plugins because each node in a Kubernetes cluster needs to have the networking configured to allow communication for the pods running on that node.

### How CNI Plugins Work as DaemonSets

1. **Installation**: When you install a CNI plugin (like Calico, Flannel, or Weave), it is typically deployed as a DaemonSet. This ensures that every node in the cluster runs an instance of the CNI plugin.

2. **Pod Networking**: Each time a pod is created on a node, the Kubelet (the agent running on each node) interacts with the CNI plugin to set up the network for that pod. The CNI plugin will create network interfaces and assign IP addresses as necessary.

3. **Node Configuration**: The CNI DaemonSet typically includes components that set up networking for both pods and the host network, ensuring proper routing and communication between them.

### Benefits of Using DaemonSets for CNI

- **Consistent Configuration**: By running the CNI plugin on every node, you ensure that all nodes have the same networking configuration.
- **Automatic Scaling**: As you add or remove nodes in your cluster, the DaemonSet automatically manages the deployment of the CNI plugin on those nodes.
- **Resilience**: If a node fails and is replaced, the DaemonSet ensures that the new node will have the necessary networking configured automatically.

### Example of Deploying a CNI Plugin as a DaemonSet

Here’s a basic example of what a CNI plugin like Calico might look like when deployed as a DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      containers:
      - name: calico-node
        image: calico/node:v3.22.0
        env:
        - name: CALICO_NETWORKING_BACKEND
          value: "bird"
        # Other necessary environment variables and configurations
```

### Summary

Using DaemonSets for CNI plugins in Kubernetes ensures that every node has the necessary network configurations to manage pod networking effectively. This setup is critical for enabling reliable and scalable communication in a Kubernetes cluster.
