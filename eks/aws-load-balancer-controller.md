# Install AWS Load Balancer Controller

If you are planning to deploy Ingress resources in Amazon EKS and you are reading this guideline, you are most likely considering choosing the ALB/ELB type of Ingress due to its simplicity.

ALB/ELB Ingress implements native Ingress resources, but it requires the AWS Load Balancer Controller to be installed. Its purpose is to manage AWS resources as declared in Ingress resources.

Follow the steps below to install the AWS Load Balancer Controller.

## Pre-requisites

- Have EKS cluster up and running with following policies attached to the nodegroup:
    - `AmazonEKSWorkerNodePolicy`: Grants necessary permissions for worker nodes to interact with Amazon EKS and other AWS services.
    - `AmazonEKS_CNI_Policy`: Provides permissions for the AWS VPC CNI plugin to manage network interfaces on behalf of Kubernetes pods in the EKS cluster.
    - `AmazonEC2ContainerRegistryReadOnly`: Allows read-only access to Amazon ECR for pulling container images during deployment.
    - `ElasticLoadBalancingFullAccess`: Provides full access to Elastic Load Balancing, enabling management of ALBs for Ingress resources in the EKS cluster.
- Install helm
- Install AWS CLI and configured with permissions: `iam:CreatePolicy`, `iam:AttachUserPolicy`, `iam:AttachGroupPolicy`, `iam:AttachRolePolicy`

## Installing the Controller

1. Create `AWSLoadBalancerControllerIAMPolicy` IAM policy
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

rm iam_policy.json
```

2. IAM Service Account

Next step is to create Kubernetes iamserviceaccount for the AWS Load Balancer controller and assign it a role with above created policy. It will allow the controller to create and manage load balancers in AWS.
```bash
eksctl create iamserviceaccount --cluster={cluster-name} --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::{your-aws-account-id}:policy/AWSLoadBalancerControllerIAMPolicy --approve --profile {your-profile}
```

3. Install AWS Load Balancer Controller

Controller will be installed with Helm. First, you have to add and update the eks reporitory:
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```
After the repository is updated, you can install the controller with following command:
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName={cluster-name} --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller 
```

4. Check status

You can check if the controller was sucessfully installed by running the following command:

```bash
kubectl get pod -l app.kubernetes.io/name=aws-load-balancer-controller -n kube-system
```

Output should be something like that, if you installed the controller sucessfully (typically you get one pod per node installed, to ensure high availability and redundancy):
```
NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-7cd6bf68dd-85rqc   1/1     Running   0          23d
aws-load-balancer-controller-7cd6bf68dd-zw9q8   1/1     Running   0          23d
```

## Uninstall the Controller

Uninstalling the controller is simple. Just run a helm command for uninstalling the release:

```bash
helm uninstall {release-name} -n {namespace}
```
In our case, this means:

```bash
helm uninstall aws-load-balancer-controller -n kube-system
```