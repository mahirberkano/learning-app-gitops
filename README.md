# learning-app-gitops

GitOps repository for the learning-app Kubernetes deployment. ArgoCD watches this repo and keeps the cluster in sync.

## How GitOps Works

```
This repo (desired state) ──▶ ArgoCD (controller) ──▶ Kubernetes (actual state)
```

- You change manifests here → ArgoCD applies them to the cluster
- Someone manually edits the cluster → ArgoCD reverts it (self-healing)
- You delete a resource from Git → ArgoCD deletes it from the cluster (pruning)
- CI pushes a new image tag here → ArgoCD rolls out the new version

**Git is the single source of truth. The cluster always matches this repo.**

## Repo Structure

```
learning-app-gitops/
├── root-app.yaml              # The "App of Apps" — apply this ONCE manually
├── apps/                      # ArgoCD Application manifests
│   └── learning-app.yaml      # Points to the Helm chart below
└── charts/
    └── learning-app/          # Helm chart
        ├── Chart.yaml
        ├── values.yaml        # Image tag, replicas, resources (CI updates the tag)
        └── templates/
            ├── namespace.yaml
            ├── configmap.yaml
            ├── redis.yaml
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml
```

## App of Apps Pattern

Instead of manually applying each Application, we use a hierarchy:

```
root-app (watches apps/ directory)
  └── learning-app (watches charts/learning-app/)
```

To add a new microservice:
1. Create its Helm chart in `charts/new-service/`
2. Add an Application manifest in `apps/new-service.yaml`
3. Push — root-app picks it up automatically

## Setup

### Prerequisites
- Kubernetes cluster (minikube for local, EKS for production)
- ArgoCD installed in the cluster
- Ingress controller (nginx)

### First-time Setup

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd --wait

# 2. Apply the root app (only manual step ever)
kubectl apply -f root-app.yaml

# 3. Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 9090:443 &
# Get password:
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
# Open https://localhost:9090 — login: admin / <password>
```

After this, everything is automated. Push to this repo = deploy to cluster.

## values.yaml — What CI Changes vs What You Change

```yaml
image:
  repository: mahirberkan/learning-app
  tag: "a3f9b2c"       # ← CI updates this (git SHA of source repo)
  pullPolicy: Always

replicas: 4             # ← You change this (or HPA manages it)

resources:              # ← You change this
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 128Mi
```

**Rule of thumb:**
- CI changes: `image.tag` only
- You change: everything else (replicas, resources, config, new templates)

## ArgoCD Sync Configuration

```yaml
syncPolicy:
  automated:
    prune: true       # Remove resources deleted from Git
    selfHeal: true    # Revert manual cluster edits
```

- **Polling interval:** 60 seconds (configurable in argocd-cm ConfigMap)
- **Production alternative:** GitHub webhooks for instant sync

## Useful Commands

```bash
# Check sync status
kubectl get applications -n argocd

# Force immediate sync
# (or click Refresh → Sync in ArgoCD UI)

# See what ArgoCD would apply (dry run)
helm template learning-app charts/learning-app/

# View rendered manifests
helm template learning-app charts/learning-app/ -f charts/learning-app/values.yaml
```

## Related

- [learning-app-src](https://github.com/mahirberkano/learning-app-src) — Application source code + CI pipeline
