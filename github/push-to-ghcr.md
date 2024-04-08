# How to Build & Push the image to GitHub Container Registry (ghcr.io)

These instructions will help you to build the docker image and push it to Github Packages.

## Prerequisites:
1. Install Docker
2. Configure Github PAT (Personal Access Token) with the permission to write to Packages

## Push the image to Github Packages by doing following steps:

1. Build docker image and tag it accordingly 

Tag your Docker image for the GitHub Container Registry. The format for the tag is ghcr.io/ORGANIZATION/IMAGE_NAME:TAG. Make sure to replace LOCAL_IMAGE_NAME and TAG with the name and tag of your local Docker image, and IMAGE_NAME with the name you want for your image on GitHub.
```bash
docker build -t {LOCAL_IMAGE_NAME}:{TAG} .
docker tag {LOCAL_IMAGE_NAME}:{TAG} ghcr.io/{YOUR-ORGANISATION}/{IMAGE_NAME}:{TAG}
```

2. Login to GitHub Container Registry (ghcr.io)

Authenticate with Docker using your GitHub Personal Access Token (PAT) and your GitHub username. Replace YOUR_PAT with your actual Personal Access Token and YOUR_USERNAME with your GitHub username.
```bash
echo "{YOUR_PAT}" | docker login ghcr.io -u {YOUR_USERNAME} --password-stdin
```

3. Push the Image

Push your tagged image to the GitHub Container Registry with the following command:
```bash
docker push ghcr.io/{ORGANISATION}/{IMAGE_NAME}:{TAG}
```

4. Pull the Image

To pull the image from Docker Container Registry, use the docker pull command with the name of your image on ghcr.io. Use the command:
```bash
docker pull ghcr.io/{ORGANISATION}/{IMAGE_NAME}:{TAG}
```