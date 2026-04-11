# gitops-demo — ArgoCD Multi-Cluster GitOps Repository

A demo repository showcasing ArgoCD App-of-Apps across multiple clusters, environments, and applications.

## Architecture

```
hou-sbx-01 (ArgoCD mgmt)
  └── manages ──┬── kty-dev-01   (development)
                ├── kty-prd-01   (production)
                └── kty-prd-02   (production)
```

## Clusters

| Cluster | Role | Environment |
|---------|------|-------------|
| `hou-sbx-01` | ArgoCD control plane | Management (no app workloads) |
| `kty-dev-01` | Workload | Development |
| `kty-prd-01` | Workload | Production |
| `kty-prd-02` | Workload | Production |

## Applications

| App | Chart | Registry | Dev | Prod |
|-----|-------|----------|-----|------|
| `webapp` | podinfo 6.7.1 | `ghcr.io/stefanprodan/charts` | 1 replica | 3 replicas |
| `cache` | redis 19.6.4 | `charts.bitnami.com/bitnami` | standalone | AppSet (primary + replica shards) |
| `database` | postgresql 15.5.28 | `charts.bitnami.com/bitnami` | 5Gi, no replicas | 50Gi + read replica |

## Bootstrap (one-time)

```bash
# 1. Install ArgoCD on hou-sbx-01
kubectl apply -k argocd-install/clusters/hou-sbx-01/

# 2. Wait for ArgoCD to be ready
kubectl -n argocd wait --for=condition=available deploy/argocd-server --timeout=120s

# 3. Register workload clusters (run on each workload cluster)
kubectl apply -f argocd-service-accounts/sa.yaml

# 4. Add cluster secrets to ArgoCD (see argocd-service-accounts/README.md)
# ArgoCD will then auto-sync everything else from Git
```

## Sync Hierarchy

```
LEVEL 0  kubectl apply -k argocd-install/clusters/hou-sbx-01/
           └── installs ArgoCD + root-app-of-apps

LEVEL 1  root-app-of-apps
           └── watches app-of-apps/clusters/hou-sbx-01/

LEVEL 2  app-of-apps/clusters/hou-sbx-01/
           ├── infra-apps-{cluster} → deploys namespaces to workload cluster
           └── apps-{cluster}       → creates Application CRs in ArgoCD

LEVEL 3  Individual Applications / ApplicationSets
           └── deploys webapp, cache, database to each workload cluster
```

## Adding a New App

1. Create `apps/clusters/{cluster}/{app}/` with Application CR
2. Add `- {app}/` to `apps/clusters/{cluster}/kustomization.yaml`
3. Add values to `manifests/{app}/clusters/{cluster}/values.yaml`
4. Add namespace to `secure/clusters/{cluster}/{app}/`
5. Add AppProject to `argocd-projects/base/{app}.yaml`
6. PR → merge → ArgoCD auto-syncs
