# Minikube + ArgoCD Setup Guide

Complete guide to set up Minikube locally and deploy the Node.js app using ArgoCD.

---

## 📋 Prerequisites

- Docker Desktop installed and running
- At least 4GB RAM and 2 CPUs available
- macOS (you're on Darwin)

---

## 🚀 Step 1: Install Required Tools

### Install Minikube
```bash
# Using Homebrew
brew install minikube

# Verify installation
minikube version
```

### Install kubectl (if not already installed)
```bash
brew install kubectl

# Verify installation
kubectl version --client
```

### Install ArgoCD CLI
```bash
brew install argocd

# Verify installation
argocd version --client
```

---

## 🎯 Step 2: Start Minikube

### Start Minikube with recommended settings
```bash
# Start Minikube with Docker driver
minikube start --driver=docker --cpus=4 --memory=4096

# Check status
minikube status

# Verify kubectl is pointing to minikube
kubectl config current-context
# Should output: minikube
```

### Enable useful addons
```bash
# Enable metrics server
minikube addons enable metrics-server

# Enable ingress (optional, for later use)
minikube addons enable ingress

# List all addons
minikube addons list
```

---

## 🔧 Step 3: Install ArgoCD in Minikube

### Create ArgoCD namespace and install
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Check all pods are running
kubectl get pods -n argocd
```

### Access ArgoCD UI

**Option 1: Port Forward (Recommended for local)**
```bash
# Port forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ArgoCD will be available at: https://localhost:8080
# (Keep this terminal open)
```

**Option 2: NodePort Service**
```bash
# Change service type to NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get the URL
minikube service argocd-server -n argocd --url
```

### Get ArgoCD Admin Password
```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Save this password! You'll need it to login
```

### Login to ArgoCD
```bash
# Login via CLI (use the password from above)
argocd login localhost:8080 --username admin --insecure

# Change the password (optional but recommended)
argocd account update-password
```

---

## 📦 Step 4: Prepare Your Application

### Clone the manifests repository locally
```bash
cd /Users/arup/mydevops-projects/
git clone https://github.com/arup07/node-demo-app-manifests.git
cd node-demo-app-manifests
```

### Verify all manifest files exist
```bash
ls -l
# Should see: deployment.yaml, service.yaml, configmap.yaml, secret.yaml, etc.
```

---

## 🎨 Step 5: Create ArgoCD Application

### Method 1: Using ArgoCD CLI (Recommended)
```bash
# Create the application
argocd app create node-demo-app \
  --repo https://github.com/arup07/node-demo-app-manifests.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Check application status
argocd app get node-demo-app

# Sync the application (deploy)
argocd app sync node-demo-app
```

### Method 2: Using kubectl (Alternative)
Create file: `argocd-application.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: node-demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/arup07/node-demo-app-manifests.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Then apply:
```bash
kubectl apply -f argocd-application.yaml
```

---

## 🔍 Step 6: Verify Deployment

### Check ArgoCD Application
```bash
# Watch the sync status
argocd app get node-demo-app --refresh

# View sync history
argocd app history node-demo-app
```

### Check Kubernetes Resources
```bash
# Check all resources in default namespace
kubectl get all -n default

# Check pods
kubectl get pods -n default -w

# Check services
kubectl get svc -n default

# Check secrets and configmaps
kubectl get secrets,configmaps -n default
```

### View Logs
```bash
# Node app logs
kubectl logs -l app=node-demo-app -f

# PostgreSQL logs
kubectl logs -l app=postgres -f
```

---

## 🌐 Step 7: Access Your Application

### Get the service URL
```bash
# For LoadBalancer service in Minikube, use minikube tunnel
minikube tunnel
# (Keep this terminal open)
```

**In another terminal:**
```bash
# Get the external IP
kubectl get svc node-demo-app-service

# Once you have the EXTERNAL-IP, access:
curl http://<EXTERNAL-IP>/
curl http://<EXTERNAL-IP>/users
```

### Alternative: Use NodePort
```bash
# Change service type to NodePort if LoadBalancer doesn't work
kubectl patch svc node-demo-app-service -p '{"spec": {"type": "NodePort"}}'

# Get the URL
minikube service node-demo-app-service --url

# Access the app
curl $(minikube service node-demo-app-service --url)
```

### Alternative: Port Forward
```bash
# Port forward directly to a pod
kubectl port-forward svc/node-demo-app-service 3000:80

# Access at: http://localhost:3000
```

---

## 🎯 Step 8: Test the GitOps Workflow

### Make a change to trigger deployment
```bash
cd /Users/arup/mydevops-projects/node-demo-app-manifests

# Edit deployment.yaml - change replicas from 2 to 3
sed -i '' 's/replicas: 2/replicas: 3/' deployment.yaml

# Commit and push
git add deployment.yaml
git commit -m "Scale to 3 replicas"
git push origin main

# Watch ArgoCD automatically sync the change
argocd app get node-demo-app --refresh

# Verify pods scaled to 3
kubectl get pods -l app=node-demo-app
```

---

## 🔧 Useful Commands

### Minikube Commands
```bash
# Stop Minikube
minikube stop

# Start Minikube again
minikube start

# Delete Minikube cluster (removes everything)
minikube delete

# SSH into Minikube
minikube ssh

# View Minikube dashboard
minikube dashboard

# Check Minikube IP
minikube ip
```

### ArgoCD Commands
```bash
# List all applications
argocd app list

# Get application details
argocd app get node-demo-app

# Sync application manually
argocd app sync node-demo-app

# Refresh application (check Git for changes)
argocd app sync node-demo-app --prune

# Delete application
argocd app delete node-demo-app

# View logs
argocd app logs node-demo-app
```

### Kubectl Commands
```bash
# Get all resources
kubectl get all -A

# Describe a resource
kubectl describe pod <pod-name>

# Get logs
kubectl logs <pod-name> -f

# Execute command in pod
kubectl exec -it <pod-name> -- /bin/sh

# Delete all resources in default namespace
kubectl delete all --all -n default

# Port forward
kubectl port-forward svc/<service-name> local-port:service-port
```

---

## 🐛 Troubleshooting

### ArgoCD UI not accessible
```bash
# Check ArgoCD pods
kubectl get pods -n argocd

# Restart port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Pods not starting
```bash
# Check pod status
kubectl get pods -n default

# Describe the pod
kubectl describe pod <pod-name>

# Check events
kubectl get events -n default --sort-by='.lastTimestamp'

# Check logs
kubectl logs <pod-name>
```

### Image pull errors
```bash
# If using private registry, create docker secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=arup07 \
  --docker-password=<your-password> \
  --docker-email=arup221199@gmail.com

# Add to deployment.yaml under spec.template.spec:
# imagePullSecrets:
# - name: regcred
```

### PostgreSQL not starting
```bash
# Check if PVC is bound
kubectl get pvc

# If storage class issues, use hostPath for local testing
# Edit postgres-deployment.yaml to use emptyDir instead of PVC
```

### Application out of sync
```bash
# Force sync
argocd app sync node-demo-app --force

# Refresh and hard refresh
argocd app get node-demo-app --hard-refresh
```

---

## 📊 Monitoring

### View ArgoCD Dashboard
1. Access https://localhost:8080
2. Login with admin credentials
3. View your application sync status
4. Check resource health
5. View logs and events

### View Kubernetes Dashboard
```bash
minikube dashboard
```

---

## 🎓 Next Steps

1. **Set up Ingress** - Expose apps with custom domains
2. **Add monitoring** - Install Prometheus and Grafana
3. **Set up alerts** - Configure ArgoCD notifications
4. **Multi-environment** - Create dev, staging, prod folders
5. **Helm charts** - Convert manifests to Helm for better management

---

## 🧹 Cleanup

### Delete everything and start fresh
```bash
# Delete ArgoCD application
argocd app delete node-demo-app --yes

# Delete ArgoCD
kubectl delete namespace argocd

# Delete app resources
kubectl delete all --all -n default

# Stop Minikube
minikube stop

# Delete Minikube cluster
minikube delete
```

---

## 📚 Useful Resources

- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

**Happy GitOps! 🚀**

