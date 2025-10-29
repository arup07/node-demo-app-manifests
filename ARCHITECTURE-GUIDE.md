# Architecture Guide: PVC & ArgoCD Sync

Essential diagrams explaining Persistent Volume Claims (PVC) and ArgoCD sync workflow.

---

## 📦 Table of Contents

1. [Persistent Volume Claim (PVC) Workflow](#persistent-volume-claim-pvc-workflow)
2. [ArgoCD Sync Cycle](#argocd-sync-cycle)

---

## Persistent Volume Claim (PVC) Workflow

### How PVC Works - Complete Flow

```
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: Developer Defines PVC in YAML                      │
│                                                             │
│  apiVersion: v1                                             │
│  kind: PersistentVolumeClaim                                │
│  metadata:                                                  │
│    name: postgres-pvc                                       │
│  spec:                                                      │
│    accessModes:                                             │
│      - ReadWriteOnce  # Only ONE pod can mount             │
│    resources:                                               │
│      requests:                                              │
│        storage: 1Gi   # Request 1GB of storage             │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ kubectl apply -f postgres-deployment.yaml
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: Kubernetes Creates PVC                             │
│                                                             │
│  $ kubectl get pvc                                          │
│  NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES  │
│  postgres-pvc   Pending   -        -          -             │
│                                                             │
│  Status: Pending (waiting for storage to be provisioned)    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: Storage Class Provisioner Creates PV               │
│                                                             │
│  In Minikube:                                               │
│  - storageClassName: standard (default)                     │
│  - provisioner: k8s.io/minikube-hostpath                    │
│                                                             │
│  Automatically creates a Persistent Volume (PV):            │
│  - Creates directory on host machine                        │
│  - Allocates 1Gi of space                                   │
│  - Sets up volume with ReadWriteOnce access                 │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: PV is Bound to PVC                                 │
│                                                             │
│  $ kubectl get pvc                                          │
│  NAME           STATUS   VOLUME                  CAPACITY   │
│  postgres-pvc   Bound    pvc-abc123...           1Gi        │
│                                                             │
│  $ kubectl get pv                                           │
│  NAME              CAPACITY   ACCESS    STATUS    CLAIM     │
│  pvc-abc123...     1Gi        RWO       Bound     default/  │
│                                                   postgres- │
│                                                   pvc        │
│                                                             │
│  Physical Location (Minikube):                              │
│  /tmp/hostpath-provisioner/default-postgres-pvc-abc123/     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 5: PostgreSQL Pod Mounts the PVC                      │
│                                                             │
│  In postgres-deployment.yaml:                               │
│                                                             │
│  spec:                                                      │
│    containers:                                              │
│    - name: postgres                                         │
│      volumeMounts:                                          │
│      - name: postgres-storage                               │
│        mountPath: /var/lib/postgresql/data  ← Mount here   │
│    volumes:                                                 │
│    - name: postgres-storage                                 │
│      persistentVolumeClaim:                                 │
│        claimName: postgres-pvc  ← Reference PVC             │
│                                                             │
│  When pod starts, PVC is mounted inside the container       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 6: Data Flows Between Container and PVC               │
│                                                             │
│  ┌──────────────────────────────────────┐                  │
│  │  PostgreSQL Container                │                  │
│  │                                      │                  │
│  │  Application writes data to:         │                  │
│  │  /var/lib/postgresql/data/           │                  │
│  │    ├── base/         (tables)        │                  │
│  │    ├── global/       (catalogs)      │                  │
│  │    ├── pg_wal/       (logs)          │                  │
│  │    └── pg_stat/      (stats)         │                  │
│  └──────────────┬───────────────────────┘                  │
│                 │                                           │
│                 │ Data automatically persisted              │
│                 ↓                                           │
│  ┌──────────────────────────────────────┐                  │
│  │  PVC: postgres-pvc                   │                  │
│  │  Mounted at: /var/lib/postgresql/data│                  │
│  └──────────────┬───────────────────────┘                  │
│                 │                                           │
│                 │ Maps to                                   │
│                 ↓                                           │
│  ┌──────────────────────────────────────┐                  │
│  │  PV: Physical Storage                │                  │
│  │  /tmp/hostpath-provisioner/...       │                  │
│  └──────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

---

### Data Persistence Scenario: Pod Restart

```
┌─────────────────────────────────────────────────────────────┐
│  SCENARIO: What happens when PostgreSQL pod is deleted?     │
└─────────────────────────────────────────────────────────────┘

T=0s: Normal Operation
┌──────────────────────────────────────────────────────────┐
│  Pod: postgres-885995bd9-hfdm5                 Running   │
│  PVC: postgres-pvc                             Bound     │
│  Data: 150MB of database files on disk                   │
│                                                          │
│  /var/lib/postgresql/data/                               │
│    ├── base/          (database files)                   │
│    ├── pg_wal/        (transaction logs)                 │
│    └── postgresql.conf                                   │
└──────────────────────────────────────────────────────────┘
                       │
                       │ kubectl delete pod postgres-xxxxx
                       ↓
T=1s: Pod Terminating
┌──────────────────────────────────────────────────────────┐
│  Pod: postgres-885995bd9-hfdm5          Terminating      │
│  - PostgreSQL gracefully shuts down                      │
│  - Flushes all pending writes to disk                    │
│  - Closes database connections                           │
│                                                          │
│  PVC: postgres-pvc                      Still Bound ✅   │
│  Data: 150MB still on disk                               │
└──────────────────────────────────────────────────────────┘
                       │
                       ↓
T=3s: Pod Deleted, New Pod Creating
┌──────────────────────────────────────────────────────────┐
│  Old Pod: DELETED ❌                                      │
│                                                          │
│  New Pod: postgres-885995bd9-xyz123     ContainerCreating│
│  - Kubernetes Deployment creates replacement             │
│  - Same PVC reference in spec                            │
│                                                          │
│  PVC: postgres-pvc                      Still Bound ✅   │
│  Data: 150MB waiting to be remounted                     │
└──────────────────────────────────────────────────────────┘
                       │
                       ↓
T=5s: New Pod Mounting PVC
┌──────────────────────────────────────────────────────────┐
│  New Pod: postgres-885995bd9-xyz123     Running          │
│  - Container started                                     │
│  - PVC mounted to /var/lib/postgresql/data               │
│  - PostgreSQL finds EXISTING data files ✅               │
│                                                          │
│  PVC: postgres-pvc                      Bound to new pod │
│  Data: Same 150MB, no data loss!                         │
└──────────────────────────────────────────────────────────┘
                       │
                       ↓
T=10s: Database Recovered
┌──────────────────────────────────────────────────────────┐
│  PostgreSQL Recovery Process:                            │
│  1. Reads existing database files                        │
│  2. Applies Write-Ahead Log (WAL) if needed              │
│  3. Validates data integrity                             │
│  4. Database fully operational                           │
│                                                          │
│  Result: ZERO DATA LOSS! 🎉                              │
│  - All tables intact                                     │
│  - All data preserved                                    │
│  - Ready to accept connections                           │
└──────────────────────────────────────────────────────────┘
```

---

### PVC Lifecycle Summary

```
┌──────────────────────────────────────────────────────────┐
│  PVC States                                              │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Pending  → Waiting for storage provisioning            │
│             (StorageClass creates PV)                    │
│      ↓                                                   │
│  Bound    → PVC successfully attached to PV              │
│             (Pod can now mount it)                       │
│      ↓                                                   │
│  InUse    → Pod is actively using the PVC                │
│             (Data being read/written)                    │
│      ↓                                                   │
│  Released → Pod deleted, but data still exists           │
│             (PV not yet reclaimed)                       │
│      ↓                                                   │
│  Deleted  → PVC deleted (reclaim policy determines       │
│             if data is deleted or retained)              │
│                                                          │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  Access Modes (Important!)                               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ReadWriteOnce (RWO) ← We use this for PostgreSQL       │
│  ✓ One pod can read/write at a time                     │
│  ✗ Other pods cannot mount simultaneously                │
│  Use case: Databases (prevents data corruption)          │
│                                                          │
│  ReadWriteMany (RWX)                                     │
│  ✓ Multiple pods can read/write simultaneously           │
│  Use case: Shared file systems (NFS)                     │
│                                                          │
│  ReadOnlyMany (ROX)                                      │
│  ✓ Multiple pods can read (no writes)                    │
│  Use case: Configuration files, static content           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## ArgoCD Sync Cycle

### Complete Sync Workflow

```
┌─────────────────────────────────────────────────────────────┐
│  SOURCE OF TRUTH: Git Repository                            │
│  https://github.com/arup07/node-demo-app-manifests          │
│                                                              │
│  ├── deployment.yaml          (Desired State)               │
│  ├── service.yaml                                           │
│  ├── configmap.yaml                                         │
│  ├── secret.yaml                                            │
│  ├── postgres-deployment.yaml                               │
│  └── argocd-app.yaml                                        │
│                                                              │
│  Example: deployment.yaml                                   │
│  spec:                                                      │
│    replicas: 2                   ← Desired state            │
│    image: arup07/node-demo-app:7                            │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   │ ArgoCD polls Git every 3 minutes
                   │ (configurable interval)
                   ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: ArgoCD Fetches Latest from Git                     │
│                                                              │
│  ArgoCD Application Controller:                             │
│  - Clones/pulls Git repository                              │
│  - Reads all YAML manifests                                 │
│  - Parses Kubernetes resources                              │
│  - Builds desired state model                               │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: Query Current Cluster State                        │
│                                                              │
│  kubectl get deployment node-demo-app -o yaml               │
│  kubectl get service node-demo-app-service -o yaml          │
│  kubectl get configmap node-demo-app-config -o yaml         │
│  ... (all resources)                                        │
│                                                              │
│  Example current state:                                     │
│  spec:                                                      │
│    replicas: 2                   ← Current state            │
│    image: arup07/node-demo-app:6  (older image!)            │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: Compare Desired (Git) vs Actual (Cluster)          │
│                                                              │
│  ┌──────────────────────┐    ┌──────────────────────┐      │
│  │ Git (Desired)        │    │ Cluster (Actual)     │      │
│  ├──────────────────────┤    ├──────────────────────┤      │
│  │ replicas: 2          │ =  │ replicas: 2          │ ✓    │
│  │ image: ...:7         │ ≠  │ image: ...:6         │ ✗    │
│  │ configmap: exists    │ =  │ configmap: exists    │ ✓    │
│  └──────────────────────┘    └──────────────────────┘      │
│                                                              │
│  Result: OUT OF SYNC (image version differs)                │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: Sync Decision                                      │
│                                                              │
│  Check sync policy in argocd-app.yaml:                      │
│                                                              │
│  syncPolicy:                                                │
│    automated:                                               │
│      prune: true        # Delete resources not in Git       │
│      selfHeal: true     # Auto-fix manual changes           │
│                                                              │
│  Decision: Auto-sync is ENABLED → Proceed with sync        │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 5: Apply Changes to Cluster                           │
│                                                              │
│  ArgoCD executes:                                           │
│  kubectl apply -f deployment.yaml                           │
│                                                              │
│  Kubernetes performs rolling update:                        │
│  1. Create new pod with image :7                            │
│  2. Wait for new pod to be Ready                            │
│  3. Terminate old pod with image :6                         │
│  4. Create another new pod with image :7                    │
│  5. Terminate last old pod                                  │
│                                                              │
│  Result: Gradual update, zero downtime                      │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 6: Verify Sync Success                                │
│                                                              │
│  $ kubectl get applications -n argocd                       │
│  NAME            SYNC STATUS   HEALTH STATUS                │
│  node-demo-app   Synced        Healthy          ✅          │
│                                                              │
│  Cluster state now matches Git state                        │
│  - All pods running with image :7                           │
│  - All resources in sync                                    │
└─────────────────────────────────────────────────────────────┘
```

---

### Self-Heal Feature

```
┌─────────────────────────────────────────────────────────────┐
│  SCENARIO: Manual Change to Cluster (Drift Detection)       │
└─────────────────────────────────────────────────────────────┘

T=0s: System in Sync
┌──────────────────────────────────────────────────────────┐
│  Git:     replicas: 2                                    │
│  Cluster: replicas: 2                    Status: Synced  │
└──────────────────────────────────────────────────────────┘
                       │
                       │ Someone runs manual command:
                       │ kubectl scale deployment node-demo-app --replicas=5
                       ↓
T=1s: Manual Change Applied
┌──────────────────────────────────────────────────────────┐
│  Git:     replicas: 2    (unchanged)                     │
│  Cluster: replicas: 5    (manually changed)              │
│                                                          │
│  State: DRIFT DETECTED ⚠️                                │
└──────────────────────────────────────────────────────────┘
                       │
                       │ ArgoCD detects drift within seconds
                       ↓
T=5s: Self-Heal Triggers
┌──────────────────────────────────────────────────────────┐
│  ArgoCD Application Controller:                          │
│  - Detects cluster state ≠ Git state                     │
│  - selfHeal: true (enabled)                              │
│  - Initiates automatic sync                              │
│                                                          │
│  Action: Revert cluster to match Git                     │
└──────────────────────────────────────────────────────────┘
                       │
                       │ kubectl apply -f deployment.yaml (replicas: 2)
                       ↓
T=10s: Cluster Healed
┌──────────────────────────────────────────────────────────┐
│  Git:     replicas: 2                                    │
│  Cluster: replicas: 2    (reverted back)                 │
│                                                          │
│  State: Synced ✅                                         │
│                                                          │
│  Result: Git is ALWAYS the source of truth!              │
│  Manual changes are automatically reverted               │
└──────────────────────────────────────────────────────────┘
```

---

### Auto-Prune Feature

```
┌─────────────────────────────────────────────────────────────┐
│  SCENARIO: Resource Deleted from Git                        │
└─────────────────────────────────────────────────────────────┘

T=0s: ConfigMap Exists
┌──────────────────────────────────────────────────────────┐
│  Git Repo:                                               │
│  ├── deployment.yaml                                     │
│  ├── service.yaml                                        │
│  └── configmap.yaml         ← Exists in Git              │
│                                                          │
│  Cluster:                                                │
│  ├── Deployment: node-demo-app                           │
│  ├── Service: node-demo-app-service                      │
│  └── ConfigMap: node-demo-app-config  ← Exists in cluster│
│                                                          │
│  Status: Synced ✅                                        │
└──────────────────────────────────────────────────────────┘
                       │
                       │ Developer deletes configmap.yaml from Git
                       │ git rm configmap.yaml
                       │ git commit -m "Remove unused configmap"
                       │ git push origin main
                       ↓
T=1s: Git Updated
┌──────────────────────────────────────────────────────────┐
│  Git Repo:                                               │
│  ├── deployment.yaml                                     │
│  └── service.yaml                                        │
│  (configmap.yaml deleted)    ← No longer in Git          │
│                                                          │
│  Cluster:                                                │
│  ├── Deployment: node-demo-app                           │
│  ├── Service: node-demo-app-service                      │
│  └── ConfigMap: node-demo-app-config  ← Still in cluster │
│                                                          │
│  Status: Out of Sync ⚠️                                  │
└──────────────────────────────────────────────────────────┘
                       │
                       │ ArgoCD detects (within 3 minutes or on refresh)
                       ↓
T=3min: Auto-Prune Triggers
┌──────────────────────────────────────────────────────────┐
│  ArgoCD Analysis:                                        │
│  - ConfigMap exists in cluster                           │
│  - ConfigMap NOT in Git                                  │
│  - prune: true (enabled in sync policy)                  │
│                                                          │
│  Decision: Delete ConfigMap from cluster                 │
└──────────────────────────────────────────────────────────┘
                       │
                       │ kubectl delete configmap node-demo-app-config
                       ↓
T=3min 5s: Resource Pruned
┌──────────────────────────────────────────────────────────┐
│  Git Repo:                                               │
│  ├── deployment.yaml                                     │
│  └── service.yaml                                        │
│                                                          │
│  Cluster:                                                │
│  ├── Deployment: node-demo-app                           │
│  └── Service: node-demo-app-service                      │
│  (ConfigMap deleted)         ← Removed from cluster      │
│                                                          │
│  Status: Synced ✅                                        │
│                                                          │
│  Result: Cluster matches Git perfectly                   │
└──────────────────────────────────────────────────────────┘
```

---

### Sync Policy Configuration

```yaml
# In argocd-app.yaml

syncPolicy:
  automated:
    prune: true        # Enable auto-prune
    selfHeal: true     # Enable self-heal
    allowEmpty: false  # Don't sync if repo is empty
  
  syncOptions:
    - CreateNamespace=true  # Auto-create namespace if needed
    - PruneLast=true        # Prune resources last (safer)
  
  retry:
    limit: 5            # Retry up to 5 times on failure
    backoff:
      duration: 5s      # Initial retry after 5 seconds
      factor: 2         # Double wait time on each retry
      maxDuration: 3m   # Maximum 3 minutes between retries
```

### Sync States

```
┌─────────────────────────────────────────────────────────────┐
│  ArgoCD Application States                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Synced                                                      │
│  ✅ Cluster matches Git                                      │
│  ✅ All resources deployed correctly                         │
│  ✅ No action needed                                         │
│                                                              │
│  OutOfSync                                                   │
│  ⚠️  Git ≠ Cluster                                           │
│  ⚠️  Changes detected                                        │
│  ⚠️  Sync needed                                             │
│                                                              │
│  Syncing                                                     │
│  🔄 Sync in progress                                         │
│  🔄 Applying changes                                         │
│  🔄 Wait for completion                                      │
│                                                              │
│  Unknown                                                     │
│  ❓ Cannot determine state                                   │
│  ❓ Check connectivity                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Health States                                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Healthy                                                     │
│  ✅ All pods running                                         │
│  ✅ All services accessible                                  │
│  ✅ Application functioning                                  │
│                                                              │
│  Progressing                                                 │
│  🔄 Rolling update in progress                               │
│  🔄 Pods starting                                            │
│  🔄 Wait for ready state                                     │
│                                                              │
│  Degraded                                                    │
│  ⚠️  Some pods failing                                       │
│  ⚠️  Resources not ready                                     │
│  ⚠️  Investigate issues                                      │
│                                                              │
│  Missing                                                     │
│  ❌ Resources not found                                      │
│  ❌ Deployment failed                                        │
│  ❌ Check logs                                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Check PVC Status

```bash
# View PVC
kubectl get pvc -n default

# Describe PVC (shows events and mount status)
kubectl describe pvc postgres-pvc -n default

# Check which pod is using it
kubectl get pods -n default -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumes[*].persistentVolumeClaim.claimName}{"\n"}{end}'

# Check storage usage
kubectl exec -n default deployment/postgres -- df -h /var/lib/postgresql/data
```

### Check ArgoCD Sync Status

```bash
# View all applications
kubectl get applications -n argocd

# Get detailed app status
kubectl get application node-demo-app -n argocd -o yaml

# View sync history
argocd app history node-demo-app

# Force manual sync
argocd app sync node-demo-app

# Hard refresh (re-fetch from Git)
argocd app get node-demo-app --hard-refresh
```

---

## Summary

### Key Concepts

**PVC (Persistent Volume Claim):**
- ✅ Provides persistent storage for stateful applications
- ✅ Data survives pod restarts/deletions
- ✅ ReadWriteOnce = Only one pod can mount at a time
- ✅ Essential for databases like PostgreSQL

**ArgoCD Sync:**
- ✅ Git is the single source of truth
- ✅ Automatic synchronization every 3 minutes
- ✅ Self-heal: Reverts manual cluster changes
- ✅ Auto-prune: Deletes resources removed from Git
- ✅ Zero-downtime deployments with rolling updates

**GitOps Principle:**
```
Desired State (Git) → ArgoCD → Actual State (Cluster)
                    ↑          ↓
                    └──────────┘
                   Continuous Sync
```

---

**Author:** DevOps Team  
**Last Updated:** October 2025  
**Version:** 1.0

