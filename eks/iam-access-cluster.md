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
kubectl apply -f <path-to-your-rbac-configuration-file>
```

**3. Check resources**

After applying the resources, you can check them with a command:

```bash
kubectl get clusterroles <clusterrole-name>
kubectl get rolebinding -n <namespace> <rolebinding-name>
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
        "AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:user/<SpecificUserOrGroup>"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
Replace `<AWS_ACCOUNT_ID>` with your AWS account ID, and `SpecificUserOrGroup` with the actual IAM user or group you want to grant EKS access.

After setting up the IAM role with the correct trust relationship, you will map this IAM role to Kubernetes RBAC roles as previously described, which controls what the users assuming this IAM role can access within Kubernetes.

**3. Grant permissions to Describe cluster and Access Kubernetes API**

You must also attach a Policy, or specify Inline Permission, that allows to users assuming this role to Desribe the cluster (needed for kubectl update) and access the Kubernetes API. 

Policy JSON should look like that:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "eks:AccessKubernetesApi",
                "eks:DescribeCluster"
            ],
            "Resource": "<cluster-arn>"
        }
    ]
}
```

## Step 3: Map the IAM Role to Kubernetes RBAC

To map your IAM role to the Kubernetes RBAC, you must edit the `aws-auth` ConfigMap from the kube-system namespace. This ConfigMap is used by EKS to define the mapping of which AWS IAM roles are allowed to authenticate as which Kubernetes users/groups.

**1. Retrieve current aws-auth ConfigMap**

Save the current ConfigMap to your local machine:
```bash
kubectl get cm aws-auth -n kube-system -o yaml > aws-auth.yaml
```

**2. Update the ConfigMap**

Edit the `aws-auth.yaml` file. Add your IAM role mapping under the mapRoles section. If mapRoles doesn't exist, you'll need to add it.

Here’s an example of what to add or modify (make sure to replace placeholders with your actual values):
```bash
data:
  mapRoles: |
    - rolearn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<YourIAMRoleName>
      username: user
      groups:
        - readonly-group
```
As per our examples from Step 1, <YourIAMRoleName> would be `eks-readonly-user`

**3. Apply updated `aws-auth` ConfigMap**

After you updated the `aws-auth` ConfigMap as per your requirements, apply it with a command:
```bash
kubectl apply -f aws-auth.yaml
```

**4. Verify the Update**

If you applied the updated ConfigMap sucessully, you should see additional/modified information upon submitting the command:
```bash
kubectl get cm aws-auth -n kube-system -o yaml
```

You don't need the aws-auth.yaml file anymore.
```bash
rm aws-auth.yaml
```

## Step 4: Access the cluster

By following above steps, added a new IAM role mapping to the existing aws-auth ConfigMap. This will allow IAM users who assume your specified role to interact with your EKS cluster according to the permissions you've configured in Kubernetes RBAC.

**1. Configure AWS credentials**

You must configure AWS CLI profile first, and create a profile that assumes your specified Role. Assuming you have your profile set up in `~/.aws/credentials` file, it looks something like this:

```bash
[profile_name]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
```

Then you add additional profile that assumes the newly created role. Add following lines to the credentials file:

```bash
[new_profile_name]
role_arn = arn:aws:iam::<AWS_ACCOUNT_ID>:role/<role-name>
source_profile = profile_name
role_session_name = eks_readonly_session
```

Validate the credentials with:
```bash
aws sts get-caller-identity --profile <new_profile_name>
```
**2. Update kubectl config**

Before accessing the cluster, you must update your `kubectl` config, to add new context. Use following command:
```bash
aws eks --region <region> update-kubeconfig --name <cluster-name> --profile <new_profile_name> --alias <context-name>
```

**3. Switch kubectl context**

If you sucessfuly updated kubectl config, you can switch the current context to it, to be able to interact with cluster.

```bash
kubectl config use-context <context-name>
```

**4. Interact with cluster**

Finally, you can now interact with cluster. In example:
```bash
kubectl get pods
```
