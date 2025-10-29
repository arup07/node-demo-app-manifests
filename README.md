# Node Demo App - Kubernetes Manifests

This repository contains Kubernetes manifests for deploying the Node.js demo application with PostgreSQL database.

## 📦 Contents

- `deployment.yaml` - Node.js application deployment
- `service.yaml` - LoadBalancer service for external access
- `configmap.yaml` - Configuration for database connection
- `secret.yaml` - Database credentials (base64 encoded)
- `postgres-deployment.yaml` - PostgreSQL database deployment with PVC
- `namespace.yaml` - Namespace definition

## 🚀 Deployment Order

Apply the manifests in this order:

```bash
# 1. Create namespace (if needed)
kubectl apply -f namespace.yaml

# 2. Create secrets and configmaps
kubectl apply -f secret.yaml
kubectl apply -f configmap.yaml

# 3. Deploy PostgreSQL database
kubectl apply -f postgres-deployment.yaml

# 4. Deploy Node.js application
kubectl apply -f deployment.yaml

# 5. Create service
kubectl apply -f service.yaml
```

Or apply all at once:

```bash
kubectl apply -f .
```

## 🔍 Verify Deployment

```bash
# Check pods
kubectl get pods

# Check services
kubectl get svc

# Check deployments
kubectl get deployments

# Get application logs
kubectl logs -l app=node-demo-app

# Get database logs
kubectl logs -l app=postgres
```

## 🌐 Access the Application

After deployment, get the external IP:

```bash
kubectl get svc node-demo-app-service
```

Access the application:
- Home: `http://<EXTERNAL-IP>/`
- Database test: `http://<EXTERNAL-IP>/users`

## 🔐 Update Database Credentials

To update database credentials in `secret.yaml`:

```bash
# Encode new credentials
echo -n 'your-username' | base64
echo -n 'your-password' | base64

# Update the values in secret.yaml
# Then apply:
kubectl apply -f secret.yaml

# Restart deployments to pick up new secrets
kubectl rollout restart deployment/node-demo-app
kubectl rollout restart deployment/postgres
```

## 📊 ArgoCD Integration

This repository is designed to work with ArgoCD for GitOps-based deployments. 

The CI/CD pipeline automatically updates the image tag in `deployment.yaml` when new builds are pushed.

## 🛠️ Configuration

### Application Settings
- **Replicas**: 2 (for high availability)
- **Port**: 3000 (internal), 80 (external)
- **Image**: `arup07/node-demo-app:latest`

### Database Settings
- **Type**: PostgreSQL 15 Alpine
- **Storage**: 1Gi PVC
- **Replicas**: 1

### Resource Limits
**Node App:**
- CPU: 100m (request) / 200m (limit)
- Memory: 128Mi (request) / 256Mi (limit)

**PostgreSQL:**
- CPU: 250m (request) / 500m (limit)
- Memory: 256Mi (request) / 512Mi (limit)

## 📝 Notes

- The service is configured as `LoadBalancer` - change to `NodePort` or `ClusterIP` if needed
- PostgreSQL uses a PVC for data persistence
- Health checks are configured for both liveness and readiness probes
- All resources are deployed in the `default` namespace

