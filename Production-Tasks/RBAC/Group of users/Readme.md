When working with **groups of users** in Kubernetes, you can manage access in a more scalable way by using **groups** within your IAM provider (like AWS IAM) or other identity management systems. In Kubernetes, you can bind a **group** to a **ClusterRole** or **Role** just as you would for individual users. This enables you to grant access to multiple users by associating them with a specific group and applying roles and bindings to the group instead of managing users individually.

Let’s implement **User Groups + ClusterRole + ClusterRoleBinding** step by step. This method ensures that you can manage permissions for a **group of users** efficiently.

---

### **Step-by-Step Guide for User Groups + ClusterRole + ClusterRoleBinding**

---

### **Step 1: Create IAM Group and Users**

In AWS IAM, you can create **IAM groups** to manage access for multiple users.

#### **1.1: Create an IAM Group**

1. Go to **IAM** in the AWS Console.
2. Click **Groups** and then **Create New Group**.
3. Name the group, for example, `devs-group`.
4. Assign appropriate policies (for now, you can leave it blank or assign the necessary permissions for testing).
5. Click **Create Group**.

#### **1.2: Create Users and Add Them to the Group**

You can now create users and add them to the `devs-group`:

1. Go to **Users** and click **Add User**.
2. Create multiple users like `user1`, `user2`, etc.
3. For each user, in the **Group membership** step, select the `devs-group` you just created.
4. Complete the creation process.

Alternatively, you can create users and add them to the IAM group using the AWS CLI:

```bash
aws iam create-user --user-name user1
aws iam add-user-to-group --user-name user1 --group-name devs-group
```

---

### **Step 2: Update the `aws-auth` ConfigMap to Include the Group**

You need to update the **`aws-auth` ConfigMap** in Kubernetes to include the IAM **group** (`devs-group`) so users in this group can authenticate with Kubernetes.

1. Edit the **`aws-auth` ConfigMap**:

   ```bash
   kubectl -n kube-system get configmap/aws-auth -o yaml
   ```

2. Add the IAM group (`devs-group`) under `mapGroups` in the ConfigMap:

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
           - system:masters
     mapGroups: |
       - grouparn: arn:aws:iam::<account-id>:group/devs-group
         username: "dev-group-user"
         groups:
           - dev-group
   kind: ConfigMap
   metadata:
     name: aws-auth
     namespace: kube-system
   ```

3. **Apply the updated `aws-auth` ConfigMap**:

   ```bash
   kubectl apply -f aws-auth.yaml
   ```

   Now, all users in the `devs-group` will be able to authenticate and access Kubernetes, as they are mapped to the group `dev-group`.

---

### **Step 3: Create a ClusterRole**

Now that we have a group (`devs-group`), we can create a **ClusterRole** that grants access to specific cluster-wide resources.

1. **Create a ClusterRole** that defines what permissions the group will have. For example, we’ll grant `get`, `list`, and `watch` permissions on pods, services, and nodes.

   Create the `clusterrole.yaml` file:

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: dev-group-reader
   rules:
   - apiGroups: [""]
     resources: ["pods", "services", "nodes"]
     verbs: ["get", "list", "watch"]
   ```

2. **Apply the ClusterRole**:

   ```bash
   kubectl apply -f clusterrole.yaml
   ```

---

### **Step 4: Create a ClusterRoleBinding**

Now, we need to create a **ClusterRoleBinding** that will bind the **ClusterRole** (`dev-group-reader`) to the **group** (`dev-group`).

1. **Create a ClusterRoleBinding** to link the group (`dev-group`) to the `dev-group-reader` ClusterRole.

   Create the `clusterrolebinding.yaml` file:

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: dev-group-reader-binding
   subjects:
   - kind: Group
     name: "dev-group"  # The IAM group mapped in the aws-auth config
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: dev-group-reader
     apiGroup: rbac.authorization.k8s.io
   ```

2. **Apply the ClusterRoleBinding**:

   ```bash
   kubectl apply -f clusterrolebinding.yaml
   ```

---

### **Step 5: Verify Group Permissions**

After everything is set up, you should verify that users in the `devs-group` can access resources according to the permissions defined in the `dev-group-reader` ClusterRole.

1. **Configure `kubectl` for a user in the group**:

   Set up `kubectl` for a user (e.g., `user1`) from the `devs-group`:

   ```bash
   aws eks --region <region> update-kubeconfig --name <cluster-name> --profile <user-profile>
   ```

2. **Test the user’s access**:

   Test if `user1` (or any other user in the `devs-group`) can access the resources:

   ```bash
   kubectl get pods --as user1
   kubectl get nodes --as user1
   ```

   If everything is set up correctly, users in the `devs-group` will be able to list pods, nodes, and services at the cluster level as defined by the `dev-group-reader` ClusterRole.

---

### **In Summary:**
1. **Create IAM Group**: Create an IAM **group** (`devs-group`) and add users to it in AWS IAM.
2. **Update `aws-auth` ConfigMap**: Modify the `aws-auth` ConfigMap to map the IAM group to a Kubernetes group (e.g., `dev-group`).
3. **Create ClusterRole**: Define a **ClusterRole** that grants access to cluster-wide resources.
4. **Create ClusterRoleBinding**: Bind the **ClusterRole** to the **group** (`dev-group`) using a **ClusterRoleBinding**.
5. **Verify Permissions**: Test that users in the IAM group (`devs-group`) have the correct access to Kubernetes resources.

This method allows you to manage access for **multiple users** by organizing them into groups, making it easier to manage permissions at scale, especially in large organizations with many users.
