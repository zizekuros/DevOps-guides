# How to enable IAM user/role to access EKS cluster

This guideline will give you the instructions to set up RBAC (role-based access control) in your EKS cluster, and allow IAM user/role to access the cluster through it.

## Pre-requisites

- Configured kubectl context
- Authorize as a user with system:masters permissions

## Step 1: Define Kubernetes Role and RoleBinding

**1. Create RBAC configuration**

First, you have to create a Kubernetes Role (or ClusterRole if you want the permissions to be cluster-wide) that specifies read-only access to resources, and then a RoleBinding (or ClusterRoleBinding for cluster-wide roles) to link that role to a Kubernetes user or group.

Here are two examples of Roles and RoleBindings

*Example A*

```bash
# Simple Role and RoleBinding that alows reading pods, deployments, services, ingresses in the default namespace.

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default # or any specific namespace
  name: readonly-role
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["pods", "deployments", "services", "ingresses"]
  verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-rolebinding
  namespace: default # or any specific namespace
subjects:
- kind: User
  name: "eks-readonly-user" # This will be mapped to an IAM Role
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: readonly-role
  apiGroup: rbac.authorization.k8s.io
```

*Example B*

```bash
# One ClusterRole that allows read on ALL resources from ALL api groups, but not assigned to a particular namespace. Then there are two RoleBindings, that apply these rule in "stage" and "prod" namespaces.

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-clusterrole
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-binding-stage
  namespace: stage
subjects:
- kind: User
  name: "eks-readonly-user"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: readonly-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-binding-prod
  namespace: prod
subjects:
- kind: User
  name: "eks-readonly-user"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: readonly-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

**2. Apply resources**

When you have your Roles (or ClusterRoles) and RoleBindings (or ClusterRoleBindings) ready, apply them with a command:

```bash
kubectl apply -f {path-to-your-rbac-configuration-file}
```

**3. Check resources**

After applying the resources, you can check them with a command:

```bash
kubectl get clusterroles {clusterrole-name}
kubectl get rolebinding -n {namespace} {rolebinding-name}
```

## Step 2: Create an IAM Role for EKS Access

**1. Create an IAM Role in AWS**

- Navigate to the IAM service in the AWS Management Console.
- Create a new IAM role.
- Choose "AWS account" as the as Trusted entity type, and select "This account" under the "An AWS account" tab.
- Skip attaching any permission policies directly to this role since the access control will be managed via Kubernetes RBAC.
- Assign the same Role name as specified in your RBAC resources (i.e. eks-readonly-user as from Example A)

**2. Define the Trust Relationship**

- Edit the trust relationship of the IAM role to allow specific IAM users or groups to assume the role. The trust policy might look something like this:
```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:user/SpecificUserOrGroup"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
Replace `<AWS_ACCOUNT_ID>` with your AWS account ID, and `SpecificUserOrGroup` with the actual IAM user or group you want to grant EKS access.

After setting up the IAM role with the correct trust relationship, you will map this IAM role to Kubernetes RBAC roles as previously described, which controls what the users assuming this IAM role can access within Kubernetes.

## Step 3: Map the IAM Role to Kubernetes with RBAC

Work in progress..