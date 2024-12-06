Now, let's implement **User + ClusterRole + ClusterRoleBinding** step by step in Kubernetes. This is an extension of the previous steps where we give an IAM user permissions at the **cluster level** (instead of a specific namespace). In this example, we will create:

- **User** in AWS IAM
- **ClusterRole** that defines permissions at the cluster level.
- **ClusterRoleBinding** that binds the IAM user to the ClusterRole, granting them access to cluster-wide resources.

---

### **Step-by-Step Guide for User + ClusterRole + ClusterRoleBinding**

---

### **Step 1: Create IAM User in AWS**

We can reuse the **IAM User** (`k8s-user`) created in the previous example. However, you can follow these steps if you need to create a new one:

1. **Go to AWS IAM Console**:
   - Navigate to **IAM** → **Users** → **Add User**.
   - **User Name**: `k8s-user`
   - **Access Type**: Select **Programmatic Access**.
   - Attach any necessary policies or leave it blank for now (we will assign appropriate permissions later).
   - **Create User** and download the **Access Key ID** and **Secret Access Key**.

Alternatively, you can use AWS CLI to create the user:

```bash
aws iam create-user --user-name k8s-user
```

### **Step 2: Update the `aws-auth` ConfigMap in Kubernetes**

We need to allow the IAM user to authenticate to Kubernetes by updating the `aws-auth` ConfigMap.

1. **Edit the `aws-auth` ConfigMap**:

   ```bash
   kubectl -n kube-system get configmap/aws-auth -o yaml
   ```

   This will show the current configuration.

2. **Update the ConfigMap**:

   Add the new IAM user (`k8s-user`) under `mapUsers`:

   Example `aws-auth.yaml`:

   ```yaml
   apiVersion: v1
   data:
     mapRoles: |
       - rolearn: arn:aws:iam::<account-id>:role/<role-name>
         username: <role-username>
         groups:
           - system:masters
     mapUsers: |
       - userarn: arn:aws:iam::<account-id>:user/k8s-user
         username: k8s-user
         groups:
           - system:masters  # Or any group, like 'view' for read-only access
   kind: ConfigMap
   metadata:
     name: aws-auth
     namespace: kube-system
   ```

3. **Apply the changes**:

   ```bash
   kubectl apply -f aws-auth.yaml
   ```

   This grants the `k8s-user` the ability to authenticate to the Kubernetes API.

---

### **Step 3: Create a ClusterRole in Kubernetes**

The **ClusterRole** grants permissions to access resources at the cluster level, such as `nodes`, `deployments`, `services`, etc.

1. **Create a ClusterRole YAML file**. In this case, we'll create a **ClusterRole** that grants the user `get`, `list`, and `watch` permissions on cluster-wide resources.

   Create a file called `clusterrole.yaml`:

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     # Name of the cluster role
     name: cluster-admin-reader
   rules:
   - apiGroups: [""]
     resources: ["pods", "services", "nodes", "deployments"]
     verbs: ["get", "list", "watch"]
   ```

   This ClusterRole will give the user the ability to view (`get`, `list`, `watch`) resources like `pods`, `services`, `nodes`, and `deployments` across the entire cluster.

2. **Apply the ClusterRole to Kubernetes**:

   ```bash
   kubectl apply -f clusterrole.yaml
   ```

---

### **Step 4: Create a ClusterRoleBinding**

A **ClusterRoleBinding** binds a user, group, or service account to a **ClusterRole**, allowing them to access the resources defined in that role at the cluster level.

1. **Create a ClusterRoleBinding YAML file**. This binding links the IAM user (`k8s-user`) to the **ClusterRole** (`cluster-admin-reader`) we just created.

   Create a file called `clusterrolebinding.yaml`:

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: k8s-user-cluster-admin-reader
   subjects:
   - kind: User
     name: "k8s-user"  # The IAM user we created
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: cluster-admin-reader  # Name of the ClusterRole
     apiGroup: rbac.authorization.k8s.io
   ```

   This ClusterRoleBinding grants the `k8s-user` user the permissions defined in the `cluster-admin-reader` ClusterRole.

2. **Apply the ClusterRoleBinding to Kubernetes**:

   ```bash
   kubectl apply -f clusterrolebinding.yaml
   ```

---

### **Step 5: Verify User Permissions**

To verify that the IAM user (`k8s-user`) now has the appropriate cluster-wide permissions, follow these steps:

1. **Configure `kubectl` for the IAM user**:

   Use AWS CLI to update the kubeconfig for the new IAM user:

   ```bash
   aws eks --region <region> update-kubeconfig --name <cluster-name> --profile <user-profile>
   ```

   This ensures that `kubectl` can authenticate as `k8s-user` using the AWS IAM user.

2. **Test the permissions**:

   Verify that the `k8s-user` can list cluster-wide resources like pods, nodes, etc.

   ```bash
   kubectl get pods --as k8s-user
   kubectl get nodes --as k8s-user
   kubectl get deployments --as k8s-user
   ```

   If everything is configured correctly, the IAM user `k8s-user` will be able to see the resources that the `cluster-admin-reader` ClusterRole grants access to.

---

### **In Summary:**
1. **Create IAM User**: Create the IAM user (`k8s-user`) in AWS IAM.
2. **Update `aws-auth` ConfigMap**: Modify the `aws-auth` ConfigMap to allow the IAM user to authenticate to Kubernetes.
3. **Create ClusterRole**: Define a **ClusterRole** that grants access to cluster-wide resources (e.g., pods, services, nodes, deployments).
4. **Create ClusterRoleBinding**: Bind the IAM user to the **ClusterRole** using a **ClusterRoleBinding**.
5. **Verify Permissions**: Test the IAM user’s ability to access resources at the cluster level.

This is how you can assign **cluster-wide permissions** to a user in a Kubernetes cluster using **ClusterRoles** and **ClusterRoleBindings**.
