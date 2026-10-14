# GitOps Workflow using ArgoCD on Kubernetes

A hands-on GitOps project: **Git is the single source of truth**, and ArgoCD automatically keeps a local Kubernetes cluster in sync with this repository.

Built as part of the Elevate Labs DevOps internship (Project 10).

## What this project demonstrates

- Deploying ArgoCD on a local Kubernetes cluster (Minikube)
- Storing application manifests in Git and deploying them with ArgoCD
- **Auto-sync:** a Git commit updates the cluster with no `kubectl apply`
- **Self-heal:** manual changes to the cluster are reverted to match Git
- **Rollback:** a `git revert` returns the app to a previous state

## Architecture

```
Developer ──git push──▶ GitHub repo (app/*.yaml)
                              ▲
                              │ ArgoCD pulls and compares
                              │
                        ┌─────┴──────┐
                        │   ArgoCD   │──── applies ───▶ Kubernetes (namespace: demo)
                        └────────────┘                  ├─ Deployment: myapp (nginx)
                                                        └─ Service: myapp-service
```

## Repository structure

```
gitops-project/
├── app/
│   ├── deployment.yaml     # nginx Deployment (watched by ArgoCD)
│   └── service.yaml        # NodePort Service (watched by ArgoCD)
├── application.yaml        # ArgoCD Application (applied once with kubectl)
└── README.md
```

> `application.yaml` is deliberately **outside** `app/`, because ArgoCD syncs everything inside `app/`.

## Tools used

| Tool | Purpose |
|---|---|
| Minikube (Docker driver) | Local Kubernetes cluster |
| kubectl | Cluster CLI |
| ArgoCD v3.5.5 | GitOps controller and web UI |
| GitHub + Git | Source of truth |
| Docker Desktop | Container runtime for Minikube |
| nginx 1.25 | Sample application image |

## Prerequisites

- Docker Desktop (running)
- Minikube
- kubectl
- Git

## Setup

### 1. Start the cluster

```bash
minikube start --driver=docker
kubectl get nodes
```

### 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

Wait until all pods show `1/1 Running`.

> **Note:** `--server-side` is required. A plain `kubectl apply` fails on the ApplicationSet CRD with
> `metadata.annotations: Too long: may not be more than 262144 bytes`.

### 3. Open the ArgoCD UI

Run the port-forward in its own terminal and leave it open:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the admin password:

**Linux / macOS**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Windows PowerShell**
```powershell
$p = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($p))
```

Open `https://localhost:8080` (accept the self-signed certificate warning) and log in as `admin`.

### 4. Create the ArgoCD Application

Edit `repoURL` in `application.yaml` if you forked this repo, then:

```bash
kubectl apply -f application.yaml
```

ArgoCD pulls `app/` from this repo and deploys it to the `demo` namespace. The app should become **Healthy** and **Synced**.

```bash
kubectl get all -n demo
minikube service myapp-service -n demo
```

> With the Docker driver on Windows, keep the `minikube service` terminal open while viewing the page.

## Demos

### Demo 1: Git-driven update

Change `replicas: 2` to `replicas: 4` in `app/deployment.yaml`, then:

```bash
git add app/deployment.yaml
git commit -m "Scale to 4 replicas"
git push
```

ArgoCD detects the commit (polls about every 3 minutes, or click **Refresh**) and scales the Deployment to 4 pods.

### Demo 2: Self-healing

```bash
kubectl scale deployment myapp -n demo --replicas=1
kubectl get pods -n demo -w
```

ArgoCD sees the drift from Git and restores 4 replicas within seconds.

### Demo 3: Rollback

```bash
git revert HEAD --no-edit
git push
```

ArgoCD syncs the reverted state (back to 2 replicas). Every rollback is a Git commit with a full audit trail.

## Results

Deployment history from ArgoCD (all initiated by the automated sync policy):

| Commit | Message | Outcome |
|---|---|---|
| `5a273fc` | Add app manifests | Initial deploy, 2 pods |
| `d400725` | Scale to 4 replicas | 4 pods via Git |
| `81bfb36` | Revert "Scale to 4 replicas" | Rollback to 2 pods |
| `a4058c0` | Reapply "Scale to 4 replicas" | Roll forward to 4 pods |

## Key sync settings

```yaml
syncPolicy:
  automated:
    prune: true      # delete resources that are removed from Git
    selfHeal: true   # revert manual changes in the cluster
  syncOptions:
  - CreateNamespace=true
```

## Screenshots

Add your screenshots to a `screenshots/` folder and link them here:

- ArgoCD dashboard (Healthy, Synced)
- Resource tree with 4 pods
- Self-heal terminal output
- App Details (repo, path, image)
- History and rollback

```markdown
![Resource tree](screenshots/resource-tree.png)
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `Too long: may not be more than 262144 bytes` | Use `kubectl apply --server-side --force-conflicts` |
| `unable to forward port because pod is not running` | Wait until pods are `Running`: `kubectl get pods -n argocd -w` |
| `localhost refused to connect` | The port-forward stopped. Re-run it and keep that terminal open |
| `argocd-initial-admin-secret not found` | ArgoCD pods are not ready yet. Wait and retry |
| Commit says "nothing to commit" | Save the file (Ctrl + S) before `git add` |

## Cleanup

```bash
kubectl delete -f application.yaml
kubectl delete namespace argocd demo
minikube stop
```

## Resume line

*Implemented GitOps pipeline using ArgoCD and Kubernetes.*

## Author

Arepalli Naga Swaroop
