# Monitoring and Logging

I set up monitoring using CloudWatch since the app runs on AWS.

## What I Configured

### Container Insights

I enabled the CloudWatch Observability addon on the EKS cluster. This automatically collects metrics like CPU usage, memory, network traffic, disk usage and pod restart counts from all the nodes and pods.

To see these, go to AWS Console -> CloudWatch -> Container Insights -> pick the cluster.

### Control Plane Logs

I turned on logging for the EKS control plane. This logs things like API requests, authentication events, scheduler decisions etc. Useful for debugging cluster-level issues.

These logs show up in CloudWatch under the log group: `/aws/eks/streaming-app-cluster/cluster`

I enabled all 5 log types:
- API server logs
- Audit logs
- Authenticator logs
- Controller manager logs
- Scheduler logs

### Alarms

I set up two alarms:
- **EKS-HighCPU-StreamingApp** - fires when average CPU goes above 80% for 10 minutes
- **EKS-HighMemory-StreamingApp** - fires when average memory goes above 80% for 10 minutes

These can be connected to SNS later for email/Slack notifications.

## Checking Logs from Terminal

I mostly used kubectl for quick debugging:

```bash
# see logs from a pod
kubectl logs <pod-name>

# follow logs live
kubectl logs -f <pod-name>

# see logs for all frontend pods at once
kubectl logs -l app=frontend
```

## Checking Resource Usage

```bash
# node level CPU and memory
kubectl top nodes

# pod level
kubectl top pods

# recent events (helpful when pods won't start)
kubectl get events --sort-by='.lastTimestamp'
```

## Troubleshooting

When something wasn't working, these are the things I checked:

- Pod won't start -> `kubectl describe pod <name>` to see events and error messages
- App returning errors -> `kubectl logs <name>` to check application logs
- Can't reach the app -> `kubectl get svc` to make sure the LoadBalancer has an external IP
- Image pull errors -> usually means ECR login expired or wrong image URI in values.yaml
