# Deployment

This covers how the CI/CD pipeline works and how the app gets deployed to EKS.

## CI/CD with Jenkins

The whole flow is: I push code to GitHub -> webhook notifies Jenkins -> Jenkins runs the pipeline -> images get built and pushed to ECR.

The pipeline is defined in the `Jenkinsfile` at the root of the repo. It has these stages:

1. **Checkout** - pulls the latest code from GitHub
2. **Login to ECR** - authenticates Docker with our ECR registry
3. **Build Docker Images** - builds all 5 images in parallel (frontend, auth, streaming, admin, chat)
4. **Tag & Push** - tags each image with the ECR URI and pushes them

I set up a GitHub webhook pointing to `http://<JENKINS_IP>:8080/github-webhook/` so builds trigger automatically on every push to main.

## ECR Repos

All images go to these ECR repos in ap-south-1:

- streaming-frontend - the React app served through Nginx
- streaming-auth - handles login/signup
- streaming-streaming - video streaming logic
- streaming-admin - admin panel backend
- streaming-chat - real-time chat

## Deploying to EKS with Helm

The Helm chart is in `helm/streaming-app/`. It creates deployments and services for all 6 components (5 services + MongoDB).

To deploy:
```bash
helm install streaming-app ./helm/streaming-app
```

To update after pushing new images:
```bash
helm upgrade streaming-app ./helm/streaming-app
```

To rollback if something breaks:
```bash
helm rollback streaming-app 1
```

## What Gets Created in the Cluster

- frontend - 2 pods, exposed through a LoadBalancer on port 80
- auth - 2 pods, ClusterIP on port 3001
- streaming - 2 pods, ClusterIP on port 3002
- admin - 2 pods, ClusterIP on port 3003
- chat - 2 pods, ClusterIP on port 3004
- mongo - 1 pod, ClusterIP on port 27017

Only the frontend is publicly accessible. All backend services are internal.

## Accessing the App

```bash
kubectl get svc frontend-service
```

Copy the EXTERNAL-IP and open it in the browser. It takes a couple minutes for the DNS to start working after the first deploy.

## Some Useful Commands I Used

```bash
# see what's running
kubectl get pods

# check logs if something isn't working
kubectl logs <pod-name>

# if you need more replicas
kubectl scale deployment frontend --replicas=3

# restart pods after a config change
kubectl rollout restart deployment frontend
```
