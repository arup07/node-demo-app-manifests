# Complete Deployment Guide: Minikube to ArgoCD

A comprehensive step-by-step guide to deploy the Node.js + PostgreSQL application on Minikube using ArgoCD for GitOps.

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Step 1: Install Required Tools](#step-1-install-required-tools)
4. [Step 2: Configure and Start Minikube](#step-2-configure-and-start-minikube)
5. [Step 3: Install ArgoCD](#step-3-install-argocd)
6. [Step 4: Access ArgoCD UI](#step-4-access-argocd-ui)
7. [Step 5: Deploy Application with ArgoCD](#step-5-deploy-application-with-argocd)
8. [Step 6: Verify Deployment](#step-6-verify-deployment)
9. [Step 7: Access Your Application](#step-7-access-your-application)
10. [Step 8: Test GitOps Workflow](#step-8-test-gitops-workflow)
11. [CI/CD Pipeline Integration](#cicd-pipeline-integration)
12. [Troubleshooting](#troubleshooting)
13. [Cleanup](#cleanup)

---

## Prerequisites

Before starting, ensure you have:

- **macOS** (Darwin)
- **Docker Desktop** installed and running
- **4GB RAM** and **2 CPUs** available for Minikube
- **Git** installed
- **Internet connection** for downloading images
- Basic knowledge of Kubernetes and Git

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        CI/CD Flow                            │
│                                                              │
│  Developer → GitHub → Jenkins Pipeline                       │
│                          ↓                                   │
│                    Build Docker Image                        │
│                          ↓                                   │
│                    Push to Docker Hub                        │
│                          ↓                                   │
│              Update deployment.yaml in Git                   │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                      GitOps Flow                             │
│                                                              │
│  ArgoCD monitors Git repo (every 3 minutes)                  │
│         ↓                                                    │
│  Detects changes in manifests                                │
│         ↓                                                    │
│  Automatically syncs to Kubernetes                           │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                  Minikube Cluster                            │
│                                                              │
│  ┌──────────────────────────────────────────────┐           │
│  │  Namespace: argocd                           │           │
│  │    - ArgoCD Server                           │           │
│  │    - ArgoCD Application Controller           │           │
│  │    - ArgoCD Repo Server                      │           │
│  └──────────────────────────────────────────────┘           │
│                                                              │
│  ┌──────────────────────────────────────────────┐           │
│  │  Namespace: default                          │           │
│  │    ┌──────────────────────────────┐          │           │
│  │    │  Node.js App (2 replicas)   │          │           │
│  │    │  - Container Port: 3000      │          │           │
│  │    │  - Image: arup07/node-demo-app│         │           │
│  │    └──────────────────────────────┘          │           │
│  │                                               │           │
│  │    ┌──────────────────────────────┐          │           │
│  │    │  PostgreSQL (1 replica)      │          │           │
│  │    │  - Port: 5432                │          │           │
│  │    │  - Persistent Volume: 1Gi    │          │           │
│  │    └──────────────────────────────┘          │           │
│  │                                               │           │
│  │    Services:                                 │           │
│  │    - node-demo-app-service (LoadBalancer)    │           │
│  │    - postgres-service (ClusterIP)            │           │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Install Required Tools

### 1.1 Install Minikube

```bash
# Using Homebrew
brew install minikube

# Verify installation
minikube version
# Expected output: minikube version: v1.x.x
```

### 1.2 Install kubectl

```bash
# Using Homebrew
brew install kubectl

# Verify installation
kubectl version --client
# Expected output: Client Version: v1.x.x
```

### 1.3 Install ArgoCD CLI

```bash
# Using Homebrew
brew install argocd

# Verify installation
argocd version --client
# Expected output: argocd: v2.x.x
```

### 1.4 Verify Docker Desktop

```bash
# Check Docker is running
docker ps

# If error, start Docker Desktop application
```

---

## Step 2: Configure and Start Minikube

### 2.1 Start Minikube Cluster

```bash
# Start Minikube with Docker driver and recommended resources
minikube start --driver=docker --cpus=4 --memory=4096

# Expected output:
# ✓ minikube v1.x.x on Darwin
# ✓ Using the docker driver
# ✓ Starting control plane node minikube in cluster minikube
# ✓ Creating docker container (CPUs=4, Memory=4096MB)
# ✓ Preparing Kubernetes v1.x.x on Docker
# ✓ Done! kubectl is now configured to use "minikube" cluster
```

### 2.2 Verify Minikube Status

```bash
# Check cluster status
minikube status

# Expected output:
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured
```

### 2.3 Verify kubectl Context

```bash
# Check current context
kubectl config current-context
# Expected output: minikube

# Test cluster connectivity
kubectl get nodes
# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.x.x
```

### 2.4 Enable Useful Addons

```bash
# Enable metrics server for resource monitoring
minikube addons enable metrics-server

# Enable ingress (optional, for future use)
minikube addons enable ingress

# List all addons
minikube addons list
```

### 2.5 Get Minikube Information

```bash
# Get Minikube IP
minikube ip
# Note this IP (e.g., 192.168.49.2)

# Open Kubernetes dashboard (optional)
minikube dashboard
# This opens the dashboard in your browser
```

---

## Step 3: Install ArgoCD

### 3.1 Create ArgoCD Namespace

```bash
# Create dedicated namespace for ArgoCD
kubectl create namespace argocd

# Verify namespace creation
kubectl get namespaces
# Should see 'argocd' in the list
```

### 3.2 Install ArgoCD

```bash
# Install ArgoCD using official manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# This installs:
# - ArgoCD API Server
# - ArgoCD Repository Server
# - ArgoCD Application Controller
# - ArgoCD Dex Server (authentication)
# - ArgoCD Redis (caching)
# - ArgoCD Notifications Controller
```

### 3.3 Wait for ArgoCD to be Ready

```bash
# Wait for all deployments to be ready (this takes 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Check all ArgoCD pods
kubectl get pods -n argocd

# Expected output (all should be Running):
# NAME                                                READY   STATUS    RESTARTS   AGE
# argocd-application-controller-0                     1/1     Running   0          2m
# argocd-applicationset-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
# argocd-dex-server-xxxxxxxxxx-xxxxx                  1/1     Running   0          2m
# argocd-notifications-controller-xxxxxxxxxx-xxxxx    1/1     Running   0          2m
# argocd-redis-xxxxxxxxxx-xxxxx                       1/1     Running   0          2m
# argocd-repo-server-xxxxxxxxxx-xxxxx                 1/1     Running   0          2m
# argocd-server-xxxxxxxxxx-xxxxx                      1/1     Running   0          2m
```

---

## Step 4: Access ArgoCD UI

### 4.1 Expose ArgoCD Server

**Method 1: NodePort (Recommended for Minikube on macOS)**

```bash
# Patch ArgoCD server service to NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Verify the service type changed
kubectl get svc argocd-server -n argocd
# Expected output:
# NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
# argocd-server   NodePort   10.109.85.230   <none>        80:32214/TCP,443:30939/TCP
```

**Method 2: Port Forward (Alternative)**

```bash
# Start port-forward (keep terminal open)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ArgoCD will be accessible at: https://localhost:8080
```

### 4.2 Get ArgoCD Access URL

```bash
# Get the ArgoCD URL using minikube service
minikube service argocd-server -n argocd --url

# Expected output:
# http://127.0.0.1:64075
# http://127.0.0.1:64076
# ❗ Because you are using a Docker driver on darwin, the terminal needs to be open to run it.

# Keep this terminal open and note the first URL
```

### 4.3 Get ArgoCD Admin Password

```bash
# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Example output: a1b2c3d4e5f6g7h8
# IMPORTANT: Save this password!
```

### 4.4 Login to ArgoCD UI

1. Open browser and navigate to the URL from step 4.2
   - Example: `http://127.0.0.1:64075`

2. Login with credentials:
   - **Username:** `admin`
   - **Password:** (from step 4.3)

3. You should see the ArgoCD dashboard (empty initially)

### 4.5 Login to ArgoCD CLI (Optional)

```bash
# Login via CLI using the URL from minikube service
argocd login 127.0.0.1:64075 --username admin --insecure

# Enter the password when prompted

# Verify login
argocd account list
```

---

## Step 5: Deploy Application with ArgoCD

### 5.1 Review the Application Manifest

The ArgoCD application is defined in `argocd-app.yaml`:

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
      prune: true        # Delete resources not in Git
      selfHeal: true     # Auto-sync when drift detected
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Key Configuration:**
- **repoURL:** Points to your GitHub manifest repository
- **path:** `.` (root of the repository)
- **destination:** `default` namespace
- **automated sync:** Enabled with auto-prune and self-heal
- **sync policy:** Will automatically deploy changes from Git

### 5.2 Apply the ArgoCD Application

```bash
# Navigate to manifests directory
cd /Users/arup/mydevops-projects/node-demo-app-manifests

# Apply the ArgoCD application manifest
kubectl apply -f argocd-app.yaml

# Expected output:
# application.argoproj.io/node-demo-app created (or unchanged)
```

### 5.3 Verify Application Creation

```bash
# Check if application was created
kubectl get applications -n argocd

# Expected output:
# NAME            SYNC STATUS   HEALTH STATUS
# node-demo-app   Unknown       Healthy
```

### 5.4 Trigger Initial Sync

```bash
# Trigger ArgoCD to sync the application
kubectl -n argocd patch app node-demo-app --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'

# Or using ArgoCD CLI:
argocd app sync node-demo-app

# Expected output:
# application.argoproj.io/node-demo-app patched
```

### 5.5 Monitor Deployment Progress

```bash
# Watch application sync status
kubectl get applications -n argocd -w
# Press Ctrl+C to stop watching

# Watch pods being created
kubectl get pods -n default -w
# Press Ctrl+C to stop watching

# Using ArgoCD CLI:
argocd app get node-demo-app
```

---

## Step 6: Verify Deployment

### 6.1 Check Application Status

```bash
# Check ArgoCD application status
kubectl get applications -n argocd

# Expected output:
# NAME            SYNC STATUS   HEALTH STATUS
# node-demo-app   Synced        Healthy
```

### 6.2 Verify All Pods are Running

```bash
# List all pods in default namespace
kubectl get pods -n default

# Expected output:
# NAME                            READY   STATUS    RESTARTS   AGE
# node-demo-app-9c7f55bd5-xxxxx   1/1     Running   0          2m
# node-demo-app-9c7f55bd5-xxxxx   1/1     Running   0          2m
# postgres-885995bd9-xxxxx        1/1     Running   0          2m
```

### 6.3 Check All Resources

```bash
# View all resources deployed
kubectl get all -n default

# Expected output includes:
# - 2 node-demo-app pods
# - 1 postgres pod
# - node-demo-app-service (LoadBalancer)
# - postgres-service (ClusterIP)
# - deployments for both apps
# - replicasets
```

### 6.4 Check Services

```bash
# List all services
kubectl get svc -n default

# Expected output:
# NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)
# node-demo-app-service   LoadBalancer   10.105.227.167   <pending>     80:31470/TCP
# postgres-service        ClusterIP      10.96.15.68      <none>        5432/TCP
```

### 6.5 Check ConfigMaps and Secrets

```bash
# Verify ConfigMap was created
kubectl get configmap node-demo-app-config -n default -o yaml

# Verify Secret was created (don't display values)
kubectl get secret node-demo-app-secret -n default
```

### 6.6 Check Persistent Volume

```bash
# Check if PVC was created for PostgreSQL
kubectl get pvc -n default

# Expected output:
# NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES
# postgres-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO
```

### 6.7 View Pod Logs

```bash
# View Node.js app logs
kubectl logs -l app=node-demo-app -n default --tail=50

# Expected to see:
# ✅ Server running on port 3000

# View PostgreSQL logs
kubectl logs -l app=postgres -n default --tail=50

# Expected to see PostgreSQL startup messages
```

---

## Step 7: Access Your Application

### 7.1 Setup Port Forwarding

```bash
# Forward local port 3000 to service port 80
kubectl port-forward svc/node-demo-app-service 3000:80 -n default

# Expected output:
# Forwarding from 127.0.0.1:3000 -> 3000
# Forwarding from [::1]:3000 -> 3000

# Keep this terminal open!
```

**Why Port Forwarding?**
- Kubernetes services run in an isolated network
- Minikube on macOS with Docker driver requires port-forward or minikube service
- Port-forward creates a tunnel from localhost to the cluster
- Format: `kubectl port-forward svc/<service-name> <local-port>:<service-port>`

### 7.2 Test the Application

**Open a new terminal and test:**

```bash
# Test home endpoint
curl http://localhost:3000/

# Expected output:
# 🚀 Node.js + PostgreSQL Demo App is running!

# Test database endpoint
curl http://localhost:3000/users

# Expected output:
# {"message":"DB connected!","time":"2025-10-29T20:11:01.246Z"}
```

### 7.3 Access in Browser

1. Open your browser
2. Navigate to: `http://localhost:3000`
3. You should see: "🚀 Node.js + PostgreSQL Demo App is running!"
4. Navigate to: `http://localhost:3000/users`
5. You should see database connection with current timestamp

### 7.4 Alternative Access Methods

**Method 1: Minikube Service (Alternative to port-forward)**

```bash
# Get service URL
minikube service node-demo-app-service --url -n default

# Example output: http://127.0.0.1:64078
# Keep terminal open and use this URL
```

**Method 2: Minikube Tunnel (For LoadBalancer)**

```bash
# In one terminal, start minikube tunnel
minikube tunnel
# May require sudo password

# In another terminal, check external IP
kubectl get svc node-demo-app-service -n default
# EXTERNAL-IP should show a real IP instead of <pending>

# Access using that IP
curl http://<EXTERNAL-IP>
```

---

## Step 8: Test GitOps Workflow

This demonstrates the power of GitOps: Git is the single source of truth!

### 8.1 View Application in ArgoCD UI

1. Go to ArgoCD UI: `http://127.0.0.1:64075` (or your URL)
2. Click on `node-demo-app` application
3. You'll see a visual representation of all resources:
   - Deployments
   - Pods
   - Services
   - ConfigMaps
   - Secrets
   - PVC

### 8.2 Make a Change in Git

```bash
# Navigate to manifests repository
cd /Users/arup/mydevops-projects/node-demo-app-manifests

# Edit deployment.yaml to scale to 3 replicas
sed -i '' 's/replicas: 2/replicas: 3/' deployment.yaml

# Verify the change
grep "replicas:" deployment.yaml
# Should show: replicas: 3

# Commit and push
git add deployment.yaml
git commit -m "Scale Node.js app to 3 replicas"
git push origin main
```

### 8.3 Watch ArgoCD Auto-Deploy

```bash
# ArgoCD checks Git every 3 minutes by default
# Or manually trigger sync:
argocd app sync node-demo-app

# Watch pods scale up
kubectl get pods -n default -w

# You should see a third pod being created
# NAME                            READY   STATUS    RESTARTS   AGE
# node-demo-app-9c7f55bd5-xxxxx   1/1     Running   0          5m
# node-demo-app-9c7f55bd5-xxxxx   1/1     Running   0          5m
# node-demo-app-9c7f55bd5-xxxxx   0/1     Pending   0          0s  ← New pod
# node-demo-app-9c7f55bd5-xxxxx   1/1     Running   0          30s ← Running
```

### 8.4 Test Self-Healing

ArgoCD's self-heal feature ensures cluster state matches Git.

```bash
# Manually scale down to 1 replica (simulating manual intervention)
kubectl scale deployment node-demo-app --replicas=1 -n default

# Check pods
kubectl get pods -n default
# You'll see only 1 pod

# Wait a few seconds, ArgoCD will detect drift and heal
# Within 1-2 minutes, ArgoCD will sync back to 3 replicas (from Git)

# Verify self-heal worked
kubectl get pods -n default
# You should see 3 pods again!
```

### 8.5 Test Auto-Prune

ArgoCD will delete resources that are removed from Git.

```bash
# Add a test ConfigMap
kubectl create configmap test-config --from-literal=test=value -n default

# Verify it exists
kubectl get configmap test-config -n default

# ArgoCD will detect this resource is not in Git
# Within a few minutes, it will be automatically deleted

# Check if it's still there after sync
argocd app sync node-demo-app
kubectl get configmap test-config -n default
# Error: configmaps "test-config" not found (pruned by ArgoCD)
```

---

## CI/CD Pipeline Integration

### How the Complete Workflow Works

```
1. Developer pushes code
         ↓
2. Jenkins detects change (webhook or polling)
         ↓
3. Jenkins Pipeline Stages:
   - Checkout code
   - Install dependencies
   - Run tests
   - SonarQube analysis
   - Build Docker image (arup07/node-demo-app:BUILD_NUMBER)
   - Push to Docker Hub
   - Update deployment.yaml with new image tag
   - Push changes to manifests repo
         ↓
4. ArgoCD detects change in Git (within 3 min)
         ↓
5. ArgoCD syncs changes to Kubernetes
         ↓
6. Kubernetes performs rolling update
         ↓
7. New version deployed with zero downtime!
```

### Trigger Jenkins Pipeline

1. Go to Jenkins Dashboard
2. Click on your pipeline job
3. Click "Build Now"
4. Monitor the pipeline stages
5. Once complete, Jenkins will update `deployment.yaml` with new image tag

### Monitor ArgoCD Deployment

```bash
# Watch for ArgoCD to detect the change
kubectl get applications -n argocd -w

# Once synced, watch the rolling update
kubectl get pods -n default -w

# Check the new image version
kubectl describe pod <pod-name> -n default | grep Image:
# Should show: Image: arup07/node-demo-app:<new-build-number>
```

### Verify New Version

```bash
# Port-forward if not already running
kubectl port-forward svc/node-demo-app-service 3000:80 -n default

# Test the application
curl http://localhost:3000/

# Check rollout status
kubectl rollout status deployment/node-demo-app -n default
```

---

## Troubleshooting

### ArgoCD UI Not Accessible

**Problem:** Cannot access ArgoCD UI

**Solution 1: Check ArgoCD pods**
```bash
kubectl get pods -n argocd
# All pods should be Running

# If not, check logs
kubectl logs -l app.kubernetes.io/name=argocd-server -n argocd
```

**Solution 2: Restart port-forward**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**Solution 3: Use minikube service**
```bash
minikube service argocd-server -n argocd
# Keep terminal open
```

### Application Not Syncing

**Problem:** ArgoCD shows "OutOfSync" status

**Solution 1: Manual sync**
```bash
argocd app sync node-demo-app --force
```

**Solution 2: Check repository access**
```bash
# Verify ArgoCD can access Git repo
argocd repo list

# Test repo connection
kubectl logs -l app.kubernetes.io/name=argocd-repo-server -n argocd | grep error
```

**Solution 3: Hard refresh**
```bash
argocd app get node-demo-app --hard-refresh
```

### Pods Not Starting

**Problem:** Pods stuck in Pending or CrashLoopBackOff

**Solution 1: Describe the pod**
```bash
kubectl describe pod <pod-name> -n default

# Look for:
# - Image pull errors
# - Resource constraints
# - Volume mount issues
```

**Solution 2: Check logs**
```bash
kubectl logs <pod-name> -n default
kubectl logs <pod-name> -n default --previous  # Previous container logs
```

**Solution 3: Check events**
```bash
kubectl get events -n default --sort-by='.lastTimestamp'
```

### Image Pull Errors

**Problem:** ErrImagePull or ImagePullBackOff

**Solution 1: Verify image exists**
```bash
# Check Docker Hub
docker pull arup07/node-demo-app:latest
```

**Solution 2: Check image name in deployment**
```bash
kubectl get deployment node-demo-app -n default -o yaml | grep image:
```

**Solution 3: For private images, create secret**
```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=arup07 \
  --docker-password=<your-password> \
  --docker-email=arup221199@gmail.com \
  -n default

# Add to deployment.yaml:
# imagePullSecrets:
#   - name: regcred
```

### Database Connection Issues

**Problem:** App can't connect to PostgreSQL

**Solution 1: Check PostgreSQL pod**
```bash
kubectl get pods -l app=postgres -n default
kubectl logs -l app=postgres -n default
```

**Solution 2: Verify service**
```bash
kubectl get svc postgres-service -n default
# Should show ClusterIP on port 5432
```

**Solution 3: Test connection from app pod**
```bash
kubectl exec -it <node-app-pod> -n default -- /bin/sh

# Inside pod:
apk add postgresql-client
psql -h postgres-service -U postgres -d nodeapp
```

**Solution 4: Check ConfigMap and Secret**
```bash
kubectl get configmap node-demo-app-config -n default -o yaml
kubectl get secret node-demo-app-secret -n default -o yaml
```

### Storage Issues

**Problem:** PostgreSQL PVC not binding

**Solution 1: Check PVC status**
```bash
kubectl get pvc -n default
kubectl describe pvc postgres-pvc -n default
```

**Solution 2: Check available storage classes**
```bash
kubectl get storageclass
```

**Solution 3: Use dynamic provisioning (Minikube)**
```bash
# Minikube uses hostPath provisioner by default
# Delete and recreate PVC
kubectl delete pvc postgres-pvc -n default
argocd app sync node-demo-app
```

### Port Forward Keeps Disconnecting

**Problem:** Port-forward session drops frequently

**Solution 1: Use minikube service instead**
```bash
minikube service node-demo-app-service --url -n default
```

**Solution 2: Run port-forward in screen/tmux**
```bash
# Install tmux
brew install tmux

# Start tmux session
tmux new -s portforward

# Run port-forward
kubectl port-forward svc/node-demo-app-service 3000:80 -n default

# Detach: Ctrl+B then D
# Reattach: tmux attach -t portforward
```

### Minikube Won't Start

**Problem:** Minikube fails to start

**Solution 1: Delete and restart**
```bash
minikube delete
minikube start --driver=docker --cpus=4 --memory=4096
```

**Solution 2: Check Docker**
```bash
docker ps
# Make sure Docker Desktop is running
```

**Solution 3: Clear cache**
```bash
minikube delete --all --purge
rm -rf ~/.minikube
minikube start --driver=docker --cpus=4 --memory=4096
```

---

## Cleanup

### Stop Application

```bash
# Delete ArgoCD application (deletes all app resources)
kubectl delete application node-demo-app -n argocd

# Or using ArgoCD CLI
argocd app delete node-demo-app --yes
```

### Delete ArgoCD

```bash
# Delete ArgoCD namespace (removes everything)
kubectl delete namespace argocd
```

### Stop Minikube

```bash
# Stop Minikube (preserves cluster state)
minikube stop

# Cluster can be restarted with: minikube start
```

### Delete Minikube Cluster

```bash
# Complete deletion (removes everything)
minikube delete

# This removes:
# - All containers
# - All volumes
# - All configurations
# - All cluster data
```

### Clean Everything

```bash
# Nuclear option: delete all Minikube profiles
minikube delete --all --purge

# Remove Minikube cache
rm -rf ~/.minikube

# Remove kubectl config (optional)
kubectl config delete-context minikube
```

---

## Best Practices

### 1. Git Repository Structure

```
node-demo-app-manifests/
├── base/                    # Base manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── overlays/               # Environment-specific
│   ├── dev/
│   ├── staging/
│   └── production/
└── argocd-app.yaml
```

### 2. Image Tagging Strategy

- ✅ Use specific tags: `arup07/node-demo-app:v1.2.3`
- ✅ Use build numbers: `arup07/node-demo-app:123`
- ❌ Avoid `latest` in production
- ✅ Semantic versioning for releases

### 3. Resource Limits

Always define resource requests and limits:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### 4. Health Checks

Implement proper liveness and readiness probes:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
```

### 5. Security

- ✅ Use Secrets for sensitive data
- ✅ Enable RBAC
- ✅ Scan images for vulnerabilities
- ✅ Use network policies
- ✅ Rotate secrets regularly

### 6. Monitoring

- ✅ Enable metrics-server
- ✅ Add Prometheus/Grafana
- ✅ Configure log aggregation
- ✅ Set up alerts

---

## Useful Commands Reference

### Minikube Commands

```bash
minikube status              # Check cluster status
minikube start               # Start cluster
minikube stop                # Stop cluster
minikube delete              # Delete cluster
minikube ip                  # Get cluster IP
minikube dashboard           # Open dashboard
minikube service <svc>       # Get service URL
minikube tunnel              # Enable LoadBalancer
minikube addons list         # List addons
minikube logs                # View logs
```

### kubectl Commands

```bash
kubectl get pods                     # List pods
kubectl get pods -A                  # List all pods (all namespaces)
kubectl get svc                      # List services
kubectl get deployments              # List deployments
kubectl describe pod <name>          # Pod details
kubectl logs <pod> -f                # Stream logs
kubectl exec -it <pod> -- /bin/sh    # Shell into pod
kubectl delete pod <name>            # Delete pod
kubectl scale deployment <name> --replicas=3  # Scale
kubectl rollout status deployment/<name>      # Check rollout
kubectl rollout undo deployment/<name>        # Rollback
kubectl port-forward svc/<svc> 8080:80        # Port forward
```

### ArgoCD Commands

```bash
argocd app list                  # List applications
argocd app get <name>            # Get app details
argocd app sync <name>           # Sync application
argocd app history <name>        # Sync history
argocd app rollback <name> <id>  # Rollback to revision
argocd app delete <name>         # Delete application
argocd app logs <name>           # View logs
argocd repo list                 # List repositories
argocd account list              # List accounts
```

---

## Next Steps

1. **Add Monitoring:**
   - Install Prometheus and Grafana
   - Configure dashboards
   - Set up alerts

2. **Implement Ingress:**
   - Configure ingress controller
   - Add custom domains
   - Setup TLS certificates

3. **Multi-Environment Setup:**
   - Create dev/staging/prod overlays
   - Use Kustomize or Helm
   - Implement promotion workflows

4. **Enhanced Security:**
   - Configure RBAC
   - Use Pod Security Standards
   - Implement Network Policies

5. **Backup & Disaster Recovery:**
   - Backup etcd
   - Backup persistent volumes
   - Document restore procedures

---

## Additional Resources

- **Minikube:** https://minikube.sigs.k8s.io/docs/
- **ArgoCD:** https://argo-cd.readthedocs.io/
- **Kubernetes:** https://kubernetes.io/docs/
- **kubectl:** https://kubernetes.io/docs/reference/kubectl/
- **GitOps:** https://www.gitops.tech/

---

**Congratulations!** 🎉 You now have a fully functional GitOps-based CI/CD pipeline!

**Author:** DevOps Team  
**Last Updated:** October 2025  
**Version:** 1.0

