Let's go through an **end-to-end** example of setting up a **Service Account (SA)**, Role, and RoleBinding in the **`dev` namespace** in Kubernetes, including creating a namespace from scratch, configuring the service account, assigning roles, and verifying functionality.

### **End-to-End Implementation: Service Account in the `dev` Namespace**

---

### **Step 1: Create the `dev` Namespace**

Namespaces allow you to logically partition resources in your cluster. Let's create the `dev` namespace first.

#### 1.1. Create the `dev` Namespace YAML

Create a file called `dev-namespace.yaml` to define the `dev` namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

#### 1.2. Apply the Namespace to the Cluster

```bash
kubectl apply -f dev-namespace.yaml
```

You can verify that the namespace is created by running:

```bash
kubectl get namespaces
```

You should see `dev` listed as one of the namespaces.

---

### **Step 2: Create a Service Account in the `dev` Namespace**

A **Service Account (SA)** in Kubernetes is used by workloads (pods) to interact with the Kubernetes API, typically for tasks like accessing secrets, creating resources, etc.

#### 2.1. Create the `my-service-account` YAML

Create a file called `service-account.yaml` to define the service account in the `dev` namespace.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: dev
```

#### 2.2. Apply the Service Account to the Cluster

```bash
kubectl apply -f service-account.yaml
```

You can verify the service account creation by running:

```bash
kubectl get serviceaccounts -n dev
```

You should see `my-service-account` listed.

---

### **Step 3: Create a Role in the `dev` Namespace**

A **Role** defines a set of permissions within a namespace. For this example, we’ll create a role that allows the service account to **list** and **get** pods within the `dev` namespace.

#### 3.1. Create the `read-pods` Role YAML

Create a file called `role.yaml` to define the role in the `dev` namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: read-pods
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

#### 3.2. Apply the Role to the Cluster

```bash
kubectl apply -f role.yaml
```

You can verify the role creation by running:

```bash
kubectl get roles -n dev
```

You should see `read-pods` listed.

---

### **Step 4: Create a RoleBinding to Bind the Role to the Service Account**

A **RoleBinding** is used to bind a role to a service account, granting it the permissions defined in the role.

#### 4.1. Create the `read-pods-binding` RoleBinding YAML

Create a file called `rolebinding.yaml` to define the role binding in the `dev` namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: my-service-account
    namespace: dev
roleRef:
  kind: Role
  name: read-pods
  apiGroup: rbac.authorization.k8s.io
```

#### 4.2. Apply the RoleBinding to the Cluster

```bash
kubectl apply -f rolebinding.yaml
```

You can verify the role binding by running:

```bash
kubectl get rolebindings -n dev
```

You should see `read-pods-binding` listed.

---

### **Step 5: Create a Pod that Uses the Service Account**

Now that the service account and roles are set up, let’s create a **Pod** in the `dev` namespace that will use the `my-service-account` service account to access the resources defined by the role.

#### 5.1. Create the Pod YAML

Create a file called `pod-with-sa.yaml` to define the pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: dev
spec:
  serviceAccountName: my-service-account  # Use the service account here
  containers:
    - name: my-container
      image: busybox
      command: ["sleep", "3600"]
```

#### 5.2. Apply the Pod YAML to the Cluster

```bash
kubectl apply -f pod-with-sa.yaml
```

You can verify the pod creation by running:

```bash
kubectl get pods -n dev
```

You should see the `my-pod` listed.

---

### **Step 6: Test the Service Account Permissions**

Let’s verify that the service account has the appropriate permissions.

1. **Exec into the Pod**:
   ```bash
   kubectl exec -it my-pod -n dev -- sh
   ```

2. **Check if the Service Account Can List Pods**:
   Since the service account has the `read-pods` role, it should be able to list pods in the `dev` namespace. Run the following command inside the pod:
   ```bash
   kubectl get pods -n dev
   ```

   You should see `my-pod` listed and any other pods running in the `dev` namespace.

3. **Test Permissions (Denied)**:
   If you try to perform an action that the service account doesn’t have permissions for, such as creating a new pod, it should fail with a permission denied error:
   ```bash
   kubectl run test-pod --image=nginx -n dev
   ```
   This will fail because the service account only has `get` and `list` permissions for pods, not `create` or `update` permissions.

---

### **Step 7: Clean Up Resources**

After you have tested the functionality, you can clean up the resources to avoid leaving unused resources in the cluster.

1. **Delete the Pod**:
   ```bash
   kubectl delete pod my-pod -n dev
   ```

2. **Delete the RoleBinding**:
   ```bash
   kubectl delete rolebinding read-pods-binding -n dev
   ```

3. **Delete the Role**:
   ```bash
   kubectl delete role read-pods -n dev
   ```

4. **Delete the Service Account**:
   ```bash
   kubectl delete serviceaccount my-service-account -n dev
   ```

5. **Delete the `dev` Namespace** (optional, to clean everything up):
   ```bash
   kubectl delete namespace dev
   ```

---

### **Conclusion**

In this end-to-end guide, we:
1. Created the `dev` namespace.
2. Created a **Service Account** in the `dev` namespace.
3. Created a **Role** that allows listing and getting pods.
4. Created a **RoleBinding** to link the service account to the role.
5. Created a pod that uses the service account.
6. Verified the permissions of the service account by listing pods and ensuring that unauthorized actions (like creating pods) are denied.
7. Cleaned up the resources to maintain a clean Kubernetes cluster.

This process demonstrates how to set up a **Service Account** in a specific namespace, assign it roles, and use it to control access to resources within Kubernetes. It's a useful pattern for managing permissions and isolation, especially in production environments.
