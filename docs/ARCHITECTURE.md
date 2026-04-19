# Architecture

This document explains how the whole StreamingApp system is set up and how different parts connect to each other.

## How It Works

The app is a MERN stack project split into multiple small services (microservices). I containerized each service using Docker, set up a CI/CD pipeline with Jenkins, and deployed everything on AWS EKS (Kubernetes).

Here's a rough flow of the system:

```
GitHub Repo --> (webhook) --> Jenkins on EC2 --> Builds Docker images --> Pushes to ECR
                                                                            |
                                                                            v
                                                                     EKS Cluster
                                                                    /     |     \
                                                            Frontend  Backend   MongoDB
                                                            (Nginx)   Services
                                                               |
                                                          LoadBalancer
                                                               |
                                                           End Users

CloudWatch collects metrics and logs from the EKS cluster
```

## What's Running Where

**GitHub** - I forked the original repo (https://github.com/UnpredictablePrashant/StreamingApp) to my account. All code, Dockerfiles, Jenkinsfile and Helm charts are here.

**Jenkins (on EC2)** - I set up Jenkins on a t2.medium EC2 instance. It picks up code changes through a GitHub webhook and runs the pipeline. The pipeline builds all 5 Docker images and pushes them to ECR.

**ECR** - This is where Docker images are stored. I created 5 repos:
- streaming-frontend
- streaming-auth
- streaming-streaming
- streaming-admin
- streaming-chat

**EKS Cluster** - The main deployment runs here. I used `eksctl` to create a cluster with 2 worker nodes (t3.medium). The app is deployed using Helm charts. The cluster can scale between 1-3 nodes.

**CloudWatch** - Handles monitoring. Container Insights is enabled to track CPU, memory, etc. I also set up alarms for high CPU and memory usage, and enabled control plane logging.

## Services and Ports

- Frontend (React + Nginx) - port 80, exposed via LoadBalancer
- Auth Service - port 3001, internal only (ClusterIP)
- Streaming Service - port 3002, internal only
- Admin Service - port 3003, internal only
- Chat Service - port 3004, internal only
- MongoDB - port 27017, internal only

Each backend service runs 2 replicas so if one goes down the other handles traffic. Frontend also has 2 replicas behind the LoadBalancer.

## How Traffic Flows

1. User hits the LoadBalancer URL in browser -> reaches one of the frontend pods
2. Frontend makes API calls to backend services through internal ClusterIP services
3. Backend services talk to MongoDB for data storage
4. When I push code to GitHub -> webhook triggers Jenkins -> new images get built and pushed to ECR
