# Argo CD GitOps Lab

Hands-on lab to set up Argo CD on EKS and demonstrate GitOps workflow.
Git changes automatically deploy to the cluster — no manual kubectl apply needed.

---

## What We Did

1. Created a sample nginx app (Deployment + Service) in a Git repo
2. Installed Argo CD on the EKS cluster
3. Created an Argo CD Application pointing to the Git repo
4. Pushed to GitHub — Argo CD auto-synced and deployed the app
5. Changed replicas in Git — Argo CD auto-scaled the deployment (GitOps in action)

---

## Prerequisites

- EKS cluster running (eks-nht, ap-southeast-2)
- kubectl configured
- GitHub account
- Git installed locally

---

## Folder Structure

```
argocd-lab/
├── apps/
│   └── nginx-demo/
│       ├── deployment.yaml      # nginx:1.27, 3 replicas
│       └── service.yaml         # ClusterIP on port 80
├── argocd-application.yaml      # Argo CD Application manifest
└── README.md
```

---

## Step-by-Step Instructions


### Step 1 — Install Argo CD on the Cluster

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify all pods are running:

```bash
kubectl get pods -n argocd
```

Expected output — all pods should be Running:

```
argocd-application-controller-0       1/1     Running
argocd-applicationset-controller-xxx  1/1     Running
argocd-dex-server-xxx                 1/1     Running
argocd-notifications-controller-xxx   1/1     Running
argocd-redis-xxx                      1/1     Running
argocd-repo-server-xxx                1/1     Running
argocd-server-xxx                     1/1     Running
```


### Step 2 — Get Argo CD Admin Password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Save this password. Username is `admin`.


### Step 3 — Create the Sample App Manifests

Create the folder structure:

```bash
mkdir -p argocd-lab/apps/nginx-demo
cd argocd-lab
```

Create `apps/nginx-demo/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Create `apps/nginx-demo/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  type: ClusterIP
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
```


### Step 4 — Create the Argo CD Application Manifest

Create `argocd-application.yaml` in the repo root:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/argocd-lab.git
    targetRevision: main
    path: apps/nginx-demo
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true        # Deletes resources removed from Git
      selfHeal: true     # Reverts manual changes on cluster
    syncOptions:
      - CreateNamespace=true
```

Key fields explained:
- source.repoURL    — your GitHub repo URL
- source.path       — folder containing the K8s manifests
- destination       — target cluster and namespace
- automated         — enables auto-sync (no manual apply needed)
- prune             — if you delete a manifest from Git, Argo CD deletes it from cluster
- selfHeal          — if someone manually changes something on cluster, Argo CD reverts it
- CreateNamespace   — creates the target namespace if it doesn't exist


### Step 5 — Push to GitHub

Create a new repo on GitHub: https://github.com/new
- Name: argocd-lab
- Public
- No README, no .gitignore, no license

Then push:

```bash
cd argocd-lab
git init
git add .
git commit -m "Initial commit - nginx-demo app"
git remote add origin https://github.com/<your-username>/argocd-lab.git
git branch -M main
git push -u origin main
```


### Step 6 — Apply the Argo CD Application

```bash
kubectl apply -f argocd-application.yaml
```

Check sync status:

```bash
kubectl get application nginx-demo -n argocd
```

Expected:

```
NAME         SYNC STATUS   HEALTH STATUS
nginx-demo   Synced        Healthy
```

Verify resources deployed:

```bash
kubectl get all -n demo
```

You should see 2 pods, 1 service, 1 deployment, 1 replicaset — all created automatically by Argo CD.


### Step 7 — Access the Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open browser: https://localhost:8080
- Username: admin
- Password: (from Step 2)

You'll see the nginx-demo application with all its resources visualized.


### Step 8 — Test GitOps (Change in Git = Auto Deploy)

Edit `apps/nginx-demo/deployment.yaml` — change replicas from 2 to 3:

```yaml
spec:
  replicas: 3
```

Commit and push:

```bash
git add .
git commit -m "Scale nginx-demo to 3 replicas"
git push
```

Watch the pods scale up automatically (Argo CD polls every 3 minutes):

```bash
kubectl get pods -n demo -w
```

A 3rd pod will appear without running any kubectl apply — that's GitOps!

---

## How Argo CD Works

```
Developer pushes code to Git
         |
         v
  Argo CD detects change (polls every 3 min)
         |
         v
  Compares Git state vs Cluster state
         |
         v
  If different → auto-syncs (applies changes to cluster)
```

Key concept: Git is the single source of truth.
- Want to scale up? Change replicas in Git.
- Want to rollback? Revert the Git commit.
- Someone manually changed something on cluster? selfHeal reverts it.

---

## Useful Commands

```bash
# Check Argo CD application status
kubectl get application nginx-demo -n argocd

# Describe application (detailed sync info)
kubectl describe application nginx-demo -n argocd

# Check Argo CD pods
kubectl get pods -n argocd

# Check deployed resources
kubectl get all -n demo

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Force a manual sync (instead of waiting 3 min)
kubectl patch application nginx-demo -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

---

## Troubleshooting


### 1 — Check Application Status

```bash
# Quick status
kubectl get application nginx-demo -n argocd

# Detailed — shows sync history, events, errors
kubectl describe application nginx-demo -n argocd
```

What to look for:
- SYNC STATUS: Synced = good, OutOfSync = Git and cluster don't match
- HEALTH STATUS: Healthy = good, Degraded/Progressing/Missing = problem
- Events section at the bottom — shows sync attempts and errors


### 2 — Check Argo CD Component Logs

```bash
# Application Controller — handles sync/reconciliation (most important)
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50

# Repo Server — clones Git repos, generates manifests
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=50

# API Server — serves UI and API
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=50
```

Which log to check when:
- App not syncing              → application-controller
- Git clone/auth errors        → repo-server
- UI not loading               → argocd-server


### 3 — Common Issues

| Problem | What to Check |
|---|---|
| App stuck OutOfSync | `kubectl describe application` — look at Events for error |
| Git auth failure | repo-server logs — wrong URL or private repo without credentials |
| Pods not starting | `kubectl get pods -n demo` + `kubectl describe pod <name> -n demo` |
| Sync succeeded but no change | Check if the right commit hash is in the Revision field |
| selfHeal not working | Verify `selfHeal: true` in syncPolicy |
| Namespace not created | Check `CreateNamespace=true` in syncOptions |


### 4 — Force a Manual Sync

Instead of waiting 3 minutes for Argo CD to poll:

```bash
kubectl patch application nginx-demo -n argocd --type merge -p '{"operation":{"sync":{}}}'
```


### 5 — Check Argo CD Pods Health

```bash
kubectl get pods -n argocd
```

If any pod is CrashLoopBackOff or not Running, check its logs:

```bash
kubectl logs -n argocd <pod-name>
```

---

## Cleanup

```bash
# Delete the Argo CD application (this also deletes the deployed resources due to prune)
kubectl delete application nginx-demo -n argocd

# Delete Argo CD itself
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd

# Delete demo namespace
kubectl delete namespace demo
```

---

## Lab Details

- Cluster: eks-nht
- Region: ap-southeast-2
- GitHub Repo: https://github.com/harshitjain99/argocd-lab
- Argo CD Namespace: argocd
- App Namespace: demo
