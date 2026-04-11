# ArgoCD Feature Demo

Standalone demo using nginx (Bitnami Helm chart) to walk through ArgoCD core
features. Completely independent of the multi-cluster setup in this repo.

---

## Structure

```
argocd-demo/
├── project.yaml              # AppProject scoping sources + destinations
├── apps/
│   ├── nginx-dev.yaml        # Application → kty-dev-01, 1 replica
│   └── nginx-prod.yaml       # Application → kty-prd-01, 2 replicas
└── manifests/
    ├── values-dev.yaml       # DEV values
    └── values-prod.yaml      # PROD values
```

---

## Demo Script

### Step 1 — Create the AppProject
```bash
kubectl apply -f argocd-demo/project.yaml
```
Open ArgoCD UI → Settings → Projects → **demo**
Show: source repos locked to bitnami + this git repo, destinations scoped to specific clusters/namespaces.
**Point:** Flux has no AppProject equivalent — no source or destination restrictions.

---

### Step 2 — Deploy nginx to dev
```bash
kubectl apply -f argocd-demo/apps/nginx-dev.yaml
```
App appears in ArgoCD UI as **OutOfSync** (manual sync is on).

Click **Diff** — show the exact Helm-rendered resources that will be created.
**Point:** ArgoCD shows you what will change before applying. Flux just applies.

Click **Sync** → app turns green. Show the resource tree (Deployment, Service, Pod).

---

### Step 3 — Show drift detection
```bash
kubectl scale deploy nginx -n nginx-dev --replicas=0
```
Wait ~3 minutes. ArgoCD UI shows **OutOfSync** with diff highlighting the replica change.
**Point:** GitOps violation is visible immediately. Flux shows this only in CLI.

To self-heal: uncomment `automated.selfHeal: true` in nginx-dev.yaml, push, watch it snap back.

---

### Step 4 — Deploy nginx to prod
```bash
kubectl apply -f argocd-demo/apps/nginx-prod.yaml
```
Show two apps side by side in the UI — different clusters, different replica counts, same chart version.

Notice `sync-wave: "1"` on prod — explains ordered deployment concept.
Notice `ignoreDifferences` on `/spec/replicas` — explains HPA coexistence.

---

### Step 5 — GitOps change loop
Edit `argocd-demo/manifests/values-prod.yaml` in GitHub — change `replicaCount: 2` to `replicaCount: 3`.
Commit and push. ArgoCD detects OutOfSync within 3 minutes.

Click **Diff** in UI → shows `replicas: 2 → 3` clearly.
Click **Sync** → change applied.

---

### Step 6 — Rollback
In ArgoCD UI: **demo.nginx-prod** → **History and Rollback** tab.
Click the previous revision → **Rollback**.
**Point:** One click, no git revert, no kubectl — Flux requires reverting commits.

---

## ArgoCD vs Flux — Feature Comparison

| Feature | ArgoCD | Flux |
|---------|--------|------|
| Web UI with app tree + health | Yes | No built-in UI |
| Manual sync + diff preview | Yes | Always auto-applies |
| Rollback from UI | Yes | Must revert git commit |
| AppProject source/dest scoping | Yes | No equivalent |
| Drift detection with visual diff | Yes | CLI only (`flux diff`) |
| Sync waves (ordered deploys) | Yes | No equivalent |
| ApplicationSets (multi-cluster generators) | Yes | No equivalent |
| Self-healing | Yes | Yes |

---

## How to apply values change for live demo
```bash
# In git: edit argocd-demo/manifests/values-prod.yaml
# Change replicaCount or the serverBlock message
git add argocd-demo/manifests/values-prod.yaml
git commit -m "demo: update prod nginx config"
git push origin main
# ArgoCD detects change within ~3 min (default poll interval)
```
