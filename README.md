# Fleet Management Application - Kubernetes Deployment Guide

## Overview
This directory contains Kubernetes manifests for deploying a distributed fleet management application with real-time vehicle tracking capabilities.

## Architecture
The application consists of the following microservices:

1. **fleetman-queue**: ActiveMQ message queue for inter-service communication
2. **fleetman-position-simulator**: Generates vehicle position data
3. **fleetman-position-tracker**: Consumes and processes vehicle positions
4. **fleetman-api-gateway**: API Gateway for backend services
5. **fleetman-webapp**: Web frontend for fleet visualization
6. **fleetman-mongodb**: MongoDB database for data persistence

## Prerequisites
- Kubernetes cluster (v1.19+)
- kubectl configured to access your cluster
- Sufficient cluster resources (minimum 4 CPU cores, 8GB RAM)

## Deployment Instructions

### 1. Create the Namespace
```powershell
kubectl apply -f namespace.yaml
```

### 2. Deploy MongoDB with Persistent Storage
```powershell
kubectl apply -f mongodb-pvc.yaml
kubectl apply -f mongodb-deployment.yaml
```

### 3. Deploy the Message Queue
```powershell
kubectl apply -f queue-deployment.yaml
```

### 4. Deploy Position Services
```powershell
kubectl apply -f position-simulator-deployment.yaml
kubectl apply -f position-tracker-deployment.yaml
kubectl apply -f position-tracker-service.yaml
```

### 5. Deploy API Gateway
```powershell
kubectl apply -f api-gateway-deployment.yaml
kubectl apply -f api-gateway-service.yaml
```

### 6. Deploy Web Application
```powershell
kubectl apply -f webapp-deployment.yaml
kubectl apply -f webapp-service.yaml
```

### Deploy All at Once
```powershell
kubectl apply -f .
```

## Accessing the Application

After deployment, the following services are accessible via NodePort:

- **Web Application**: http://<NODE_IP>:30080
- **API Gateway**: http://<NODE_IP>:30020
- **Position Tracker**: http://<NODE_IP>:30010

To get your node IP:
```powershell
kubectl get nodes -o wide
```

## Verification Commands

### Check all resources in the fleetman namespace
```powershell
kubectl get all -n fleetman
```

### Check pod status
```powershell
kubectl get pods -n fleetman
```

### Check service endpoints
```powershell
kubectl get services -n fleetman
```

### Check persistent volume claims
```powershell
kubectl get pvc -n fleetman
```

### View logs for a specific pod
```powershell
kubectl logs -n fleetman <pod-name>
```

### Describe a pod for troubleshooting
```powershell
kubectl describe pod -n fleetman <pod-name>
```

## Scaling

To scale a deployment:
```powershell
kubectl scale deployment <deployment-name> --replicas=<number> -n fleetman
```

Example:
```powershell
kubectl scale deployment fleetman-webapp --replicas=3 -n fleetman
```

## Resource Configuration

Each deployment includes:
- **Resource Requests**: Minimum guaranteed resources
- **Resource Limits**: Maximum allowed resources
- **Readiness Probes**: Ensures pod is ready to receive traffic
- **Liveness Probes**: Restarts unhealthy pods automatically
- **Replicas**: High availability with 2+ replicas for stateless services

## Manifest Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Creates the fleetman namespace |
| `mongodb-pvc.yaml` | PersistentVolumeClaim for MongoDB data |
| `mongodb-deployment.yaml` | MongoDB deployment and ClusterIP service |
| `queue-deployment.yaml` | ActiveMQ queue deployment and ClusterIP service |
| `position-simulator-deployment.yaml` | Position simulator deployment and ClusterIP service |
| `position-tracker-deployment.yaml` | Position tracker deployment |
| `position-tracker-service.yaml` | Position tracker NodePort service (30010) |
| `api-gateway-deployment.yaml` | API Gateway deployment |
| `api-gateway-service.yaml` | API Gateway NodePort service (30020) |
| `webapp-deployment.yaml` | Web frontend deployment |
| `webapp-service.yaml` | Web frontend NodePort service (30080) |

## Cleanup

To remove all resources:
```powershell
kubectl delete namespace fleetman
```

Or delete individual resources:
```powershell
kubectl delete -f .
```

## Troubleshooting

### Pods not starting
```powershell
kubectl describe pod <pod-name> -n fleetman
kubectl logs <pod-name> -n fleetman
```

### Service connectivity issues
```powershell
kubectl get endpoints -n fleetman
kubectl exec -it <pod-name> -n fleetman -- sh
```

### MongoDB persistence issues
```powershell
kubectl get pvc -n fleetman
kubectl describe pvc mongodb-pvc -n fleetman
```

### Check events
```powershell
kubectl get events -n fleetman --sort-by=.metadata.creationTimestamp
```

## Production Considerations

1. **Persistent Volumes**: Ensure proper storage class configuration for MongoDB PVC
2. **Resource Limits**: Adjust based on actual workload requirements
3. **Ingress**: Consider using an Ingress controller instead of NodePort for production
4. **Secrets Management**: Store sensitive data in Kubernetes Secrets
5. **Monitoring**: Implement Prometheus and Grafana for observability
6. **Backup**: Regular backup strategy for MongoDB data
7. **Security**: Implement NetworkPolicies and RBAC

## Support

For issues or questions, refer to the project documentation or contact the development team.
