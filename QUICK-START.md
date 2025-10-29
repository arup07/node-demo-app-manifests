# Quick Start - Minikube + ArgoCD

## Automated Setup

Run this one command:

```bash
./setup-minikube-argocd.sh
```

## Manual Setup

### 1. Start Minikube
```bash
minikube start --driver=docker --cpus=4 --memory=4096
```

### 2. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 3. Get ArgoCD Password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### 4. Access ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Visit: https://localhost:8080

### 5. Deploy App
```bash
kubectl apply -f argocd-app.yaml
```

### 6. Access Your App
```bash
kubectl port-forward svc/node-demo-app-service 3000:80
```
Visit: http://localhost:3000

See SETUP-GUIDE.md for detailed instructions.
