# How to set up Kubernetes cluster in Amazon EKS (Elastic Kubernetes Service)

This tutorial will help you to set up the cluster using eksctl.

## Pre-requisites 

- Install AWS CLI.
- Install kubectl.
- Install eksctl.
- Configure AWS credentials and assign the following permissions to your user: [Minimum IAM Policies](https://eksctl.io/usage/minimum-iam-policies/).
- Configure the AWS CLI credentials profile (you can also use the default profile if you prefer):
```
aws configure --profile {your-profile}
```

- When using AWS CLI, ensure to specify the profile to be used, for example:

```
aws iam get-user --profile {your-profile}
```

## Set up cluster

These instructions will help you deploy the cluster.

**1. Prepare your eksctl config** 

The cluster will be deployed with the eksctl tool, based on the provided config file. A template of such config is in the `eks-cluster.yaml` file. Make sure to adjust the values as per your requirements, including cluster name, region, sizing, and policies.

**2. Create cluster** 
```bash
eksctl create cluster -f eks-cluster.yaml --profile {your-profile}
```

**3. Wait**
The process in step 2 might take 15-20 minutes to complete, and it is completely normal. Wait and make sure not to interrupt it.

## Check cluster status

**1. Check nodes**

Check running nodes:
```bash
kubectl get nodes
```

**2. Check pods**

You should get listed at least one of each of the following pods: kube-proxy, CoreDNS, aws-node pods, by running:
```bash
kubectl get po -n kube-system
```

## Delete cluster

If you want to delete the running cluster, you can do it with the following command:

```bash
eksctl delete cluster -f eks-cluster.yaml --profile {your-profile}
```

Alternatively, you can do it with:
```bash
eksctl delete cluster --name {cluster-name} --region {cluster-region} --profile {your-profile}
```