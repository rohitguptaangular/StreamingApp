# Setup Guide

Steps I followed to set up the entire project from scratch.

## Tools I Installed

- Git (already had it)
- Docker Desktop - needed for building container images
- AWS CLI v2 - to interact with AWS from terminal
- kubectl - to manage the Kubernetes cluster
- eksctl - makes it easy to create EKS clusters
- Helm - for packaging and deploying K8s apps

On Mac, most of these can be installed with `brew install <tool-name>`.

## Getting the Code

First I forked the repo from the instructor's GitHub, then cloned my fork:

```bash
git clone https://github.com/rohitguptaangular/StreamingApp.git
cd StreamingApp
git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
```

The upstream remote is there so I can pull any updates from the original repo later using `git fetch upstream && git merge upstream/main`.

## AWS Configuration

```bash
aws configure
```

I entered my access key, secret key, set region to `ap-south-1` (Mumbai) and output format as json. To check if it worked:

```bash
aws sts get-caller-identity
```

## Creating ECR Repos

I needed a separate ECR repo for each service:

```bash
aws ecr create-repository --repository-name streaming-frontend --region ap-south-1
aws ecr create-repository --repository-name streaming-auth --region ap-south-1
aws ecr create-repository --repository-name streaming-streaming --region ap-south-1
aws ecr create-repository --repository-name streaming-admin --region ap-south-1
aws ecr create-repository --repository-name streaming-chat --region ap-south-1
```

## Building and Pushing Docker Images

Login to ECR first:
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```

Then build each image. The Dockerfiles were already in the repo so I just ran:
```bash
docker build -t streaming-frontend ./frontend
docker build -t streaming-auth ./backend/authService
docker build -t streaming-streaming -f ./backend/streamingService/Dockerfile ./backend
docker build -t streaming-admin -f ./backend/adminService/Dockerfile ./backend
docker build -t streaming-chat -f ./backend/chatService/Dockerfile ./backend
```

Note: streaming, admin and chat services need the build context set to `./backend` because their Dockerfiles reference files relative to that directory.

After building, tag and push:
```bash
ECR=<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
for svc in frontend auth streaming admin chat; do
  docker tag streaming-$svc:latest $ECR/streaming-$svc:latest
  docker push $ECR/streaming-$svc:latest
done
```

## Jenkins Setup

I spun up a t2.medium EC2 instance with Amazon Linux 2023. Had to open port 8080 in the security group for Jenkins UI.

SSH'd in and installed everything:
```bash
sudo yum install -y java-21-amazon-corretto docker git
sudo systemctl start docker && sudo systemctl enable docker

sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install -y jenkins
sudo usermod -aG docker jenkins
sudo systemctl start jenkins
```

One thing I ran into - Jenkins 2.555 needs Java 21, not Java 17. Had to install the right version and set JAVA_HOME properly before Jenkins would start.

Also installed AWS CLI on the EC2:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

In Jenkins UI, I installed Pipeline, Git, Docker Pipeline and GitHub Integration plugins. Added AWS credentials as Secret text type with IDs `aws-access-key-id` and `aws-secret-access-key`.

## EKS Cluster

```bash
eksctl create cluster \
  --name streaming-app-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

This took around 15-20 minutes. After it was done:
```bash
aws eks update-kubeconfig --name streaming-app-cluster --region ap-south-1
kubectl get nodes
```

## Deploying with Helm

```bash
helm install streaming-app ./helm/streaming-app
```

Check if everything came up:
```bash
kubectl get pods
kubectl get svc
```

The frontend-service shows an EXTERNAL-IP which is the LoadBalancer URL to access the app.

## Cleanup

When you're done, delete everything to avoid charges:
```bash
helm uninstall streaming-app
eksctl delete cluster --name streaming-app-cluster --region ap-south-1
# Also delete ECR repos and terminate the Jenkins EC2 instance
```
