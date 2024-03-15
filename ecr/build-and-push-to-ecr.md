# How to Build & Push the image to Amazon ECR (Elastic Container Registry)

These instructions will help you to create an ECR repository first. Then you are going to build the docker image and push it to ECR.

## Prerequisites:
1. Install AWS CLI
2. Configure AWS CLI (set up credentials)
3. Grant your user the permission to manage create & push to ECR repository

## Push the image to Amazon ECR by doing following steps:

1. Create repository (make sure to save repository URI)
```bash
aws ecr create-repository --repository-name {repositoryName} --region {region}
```

2. Retrieve password & authenticate through docker
```bash
aws ecr get-login-password --region {region} | docker login --username AWS --password-stdin {repositoryUri}
```

3. Build docker image and tag it accordingly (make sure to name image same as your repository name)
```bash
docker build -t {repositoryName}:{releaseTag} .
docker tag {repositoryName}:{releaseTag} {repositoryUri}:{releaseTag}
```

4. Push the image to ECR
```bash
docker push {repositoryUri}:{releaseTag}
```