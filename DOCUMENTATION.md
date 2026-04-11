# k8s-manage-app — Technical Design Document

## Table of Contents

1. [Cluster Topology](#1-cluster-topology)
2. [Repository Structure](#2-repository-structure)
3. [End-to-End Flow Diagram](#3-end-to-end-flow-diagram)
4. [Layer 0 — ArgoCD Installation](#4-layer-0--argocd-installation)
5. [Layer 1 — Root App-of-Apps Bootstrap](#5-layer-1--root-app-of-apps-bootstrap)
6. [Layer 2 — App-of-Apps (Per Cluster)](#6-layer-2--app-of-apps-per-cluster)
7. [Layer 3 — Application & ApplicationSet CRs](#7-layer-3--application--applicationset-crs)
8. [Layer 4 — Helm Values Merge](#8-layer-4--helm-values-merge)
9. [ApplicationSet — Git File Generator Pattern](#9-applicationset--git-file-generator-pattern)
10. [ArgoCD Projects (RBAC)](#10-argocd-projects-rbac)
11. [Cluster Registration](#11-cluster-registration)
12. [How to Add a New Cluster](#12-how-to-add-a-new-cluster)
13. [How to Add a New App](#13-how-to-add-a-new-app)
14. [Sync Waves & Ordering](#14-sync-waves--ordering)
15. [Environment Differences](#15-environment-differences)

---

## 1. Cluster Topology

```
┌─────────────────────────────────────────────────────────────┐
│  management cluster                                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ArgoCD (argocd namespace)                           │   │
│  │  - manages all 4 workload clusters                   │   │
│  │  - no application workloads run here                 │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────┬───────────────────────────────────────────────┘
            │ registered via argocd cluster add
     ┌──────┴──────┬──────────────┬──────────────┐
     ▼             ▼              ▼              ▼
hou-sbx-01    kty-dev-01     kty-prd-01     kty-prd-02
(sandbox)     (dev)          (prod)         (prod)
webapp        webapp         webapp         webapp
cache         cache          cache×2        cache×2
database      database       database       database
```

| Cluster | Environment | ArgoCD Destination Name | Notes |
|---------|------------|------------------------|-------|
| `management` | — | `in-cluster` | Runs ArgoCD, no workloads |
| `hou-sbx-01` | sandbox | `hou-sbx-01` | Registered workload cluster |
| `kty-dev-01` | dev | `kty-dev-01` | Registered workload cluster |
| `kty-prd-01` | prod | `kty-prd-01` | Registered workload cluster |
| `kty-prd-02` | prod | `kty-prd-02` | Registered workload cluster |

---

## 2. Repository Structure

```
k8s-manage-app/
│
├── argocd-install/                  ← How ArgoCD itself gets installed
│   ├── base/                        ← Shared patches applied to all ArgoCD instances
│   │   ├── kustomization.yaml       ← Pulls upstream install.yaml + applies patches
│   │   ├── argocd-cm.yaml           ← Annotation tracking, ignoreDifferences rules
│   │   └── argocd-rbac-cm.yaml      ← Default RBAC: readonly + admin group bindings
│   └── clusters/
│       └── management/              ← Management cluster-specific ArgoCD install
│           ├── kustomization.yaml   ← Inherits base + adds root-app-of-apps.yaml
│           ├── root-app-of-apps.yaml ← THE bootstrap Application (1 manual apply)
│           └── argocd-cm.yaml       ← Cluster-specific URL patch
│
├── app-of-apps/                     ← Layer 1: What clusters does ArgoCD manage?
│   └── clusters/
│       └── management/              ← One folder per ArgoCD instance
│           ├── kustomization.yaml   ← Lists all apps-{cluster}.yaml entries
│           ├── apps-hou-sbx-01.yaml ← Application CR pointing to apps/clusters/hou-sbx-01
│           ├── apps-kty-dev-01.yaml ← Application CR pointing to apps/clusters/kty-dev-01
│           ├── apps-kty-prd-01.yaml ← Application CR pointing to apps/clusters/kty-prd-01
│           └── apps-kty-prd-02.yaml ← Application CR pointing to apps/clusters/kty-prd-02
│
├── apps/                            ← Layer 2: What apps run on each cluster?
│   └── clusters/
│       ├── hou-sbx-01/
│       │   ├── kustomization.yaml   ← Lists webapp/, cache/, database/
│       │   ├── webapp/webapp.yaml   ← Application CR (Pattern A: direct multi-source)
│       │   ├── cache/cache.yaml     ← Application CR (Pattern A: direct)
│       │   └── database/database.yaml ← Application CR (Pattern A: direct)
│       ├── kty-dev-01/              ← Same structure as hou-sbx-01
│       ├── kty-prd-01/
│       │   ├── kustomization.yaml
│       │   ├── webapp/webapp.yaml   ← Application CR (3 replicas, ingress)
│       │   ├── cache/cache.yaml     ← ApplicationSet CR (Pattern B: git file generator)
│       │   └── database/database.yaml
│       └── kty-prd-02/              ← Same structure as kty-prd-01
│
├── manifests/                       ← Layer 3: Helm values per app per cluster
│   ├── clusters/                    ← Cross-app shared values (referenced first in every app)
│   │   ├── hou-sbx-01/values.yaml
│   │   ├── kty-dev-01/values.yaml
│   │   ├── kty-prd-01/values.yaml
│   │   └── kty-prd-02/values.yaml
│   ├── webapp/
│   │   ├── values.yaml              ← Global webapp defaults (all clusters)
│   │   └── clusters/
│   │       ├── hou-sbx-01/values.yaml
│   │       ├── kty-dev-01/values.yaml
│   │       ├── kty-prd-01/values.yaml
│   │       └── kty-prd-02/values.yaml
│   ├── cache/
│   │   ├── values.yaml              ← Global cache defaults
│   │   └── clusters/
│   │       ├── hou-sbx-01/values.yaml
│   │       ├── kty-dev-01/values.yaml
│   │       ├── kty-prd-01/
│   │       │   ├── values.yaml      ← Cluster-wide overrides (auth, etc.)
│   │       │   ├── primary/
│   │       │   │   ├── config.yaml  ← Discovered by ApplicationSet generator
│   │       │   │   └── values.yaml  ← Primary shard Helm values
│   │       │   └── replica/
│   │       │       ├── config.yaml  ← Discovered by ApplicationSet generator
│   │       │       └── values.yaml  ← Replica shard Helm values
│   │       └── kty-prd-02/          ← Same structure as kty-prd-01
│   └── database/
│       ├── values.yaml
│       └── clusters/
│           ├── hou-sbx-01/values.yaml
│           ├── kty-dev-01/values.yaml
│           ├── kty-prd-01/values.yaml
│           └── kty-prd-02/values.yaml
│
├── argocd-projects/                 ← AppProject CRs (RBAC scoping per app domain)
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── webapp.yaml
│   │   ├── cache.yaml
│   │   └── database.yaml
│   └── clusters/
│       └── management/
│           └── kustomization.yaml   ← resources: ../../base
│
└── argocd-service-accounts/
    └── sa.yaml                      ← Applied to each workload cluster before registration
```

---

## 3. End-to-End Flow Diagram

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BOOTSTRAP (one-time manual step)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

kubectl apply -k argocd-install/clusters/management/
       │
       ├─► argocd-install/base/kustomization.yaml
       │       ├─ pulls: github.com/argoproj/argo-cd/v2.11.3/manifests/install.yaml
       │       ├─ patches: argocd-cm.yaml (annotation tracking)
       │       └─ patches: argocd-rbac-cm.yaml (RBAC policy)
       │
       └─► argocd-install/clusters/management/kustomization.yaml
               ├─ inherits: ../../base
               ├─ patches: argocd-cm.yaml (URL override)
               └─ creates: root-app-of-apps (Application CR)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LEVEL 1 — root-app-of-apps watches app-of-apps/clusters/management/
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

root-app-of-apps (Application)
  source: app-of-apps/clusters/management/   ← kustomize dir
  dest:   in-cluster / argocd
       │
       ▼
app-of-apps/clusters/management/kustomization.yaml
  resources:
    - apps-hou-sbx-01.yaml
    - apps-kty-dev-01.yaml
    - apps-kty-prd-01.yaml
    - apps-kty-prd-02.yaml

  ArgoCD applies these 4 Application CRs to itself (in-cluster)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LEVEL 2 — apps-{cluster} watches apps/clusters/{cluster}/
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

apps-kty-prd-01 (Application)
  source: apps/clusters/kty-prd-01/   ← kustomize dir
  dest:   in-cluster / argocd
       │
       ▼
apps/clusters/kty-prd-01/kustomization.yaml
  resources: webapp/ cache/ database/
       │
       ├─► apps/clusters/kty-prd-01/webapp/webapp.yaml      (Application CR)
       ├─► apps/clusters/kty-prd-01/cache/cache.yaml         (ApplicationSet CR)
       └─► apps/clusters/kty-prd-01/database/database.yaml   (Application CR)

  ArgoCD applies these CRs to itself — they describe workloads for kty-prd-01

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LEVEL 3 — Individual Applications deploy Helm charts to workload clusters
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

kty-prd-01.webapp (Application)
  sources:
    [1] repoURL: ghcr.io/stefanprodan/charts  chart: podinfo  version: 6.7.1
        helm.valueFiles:
          - $values/manifests/clusters/kty-prd-01/values.yaml
          - $values/manifests/webapp/values.yaml
          - $values/manifests/webapp/clusters/kty-prd-01/values.yaml
    [2] repoURL: github.com/deegupt4/k8s-manage-app  ref: values   ← $values source
  dest: kty-prd-01 / webapp namespace
       │
       ▼
  ArgoCD fetches podinfo chart from ghcr.io
  ArgoCD fetches value files from Git repo using ref: values ($values)
  Merges 3 value files (order matters — later files win)
  Runs helm template → applies manifests to kty-prd-01
```

---

## 4. Layer 0 — ArgoCD Installation

### How kustomize installs ArgoCD

```
argocd-install/
├── base/
│   └── kustomization.yaml          ← STEP 1: pulls upstream + patches
└── clusters/management/
    └── kustomization.yaml          ← STEP 2: inherits base + cluster specifics
```

**`argocd-install/base/kustomization.yaml`:**
```yaml
resources:
  - https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.3/manifests/install.yaml
  #  ↑ fetched at apply time from GitHub — ~8000 lines of K8s manifests
  #    includes: Deployments, Services, CRDs, RBAC, ConfigMaps for all ArgoCD components
patches:
  - path: argocd-cm.yaml        ← overlays the argocd-cm ConfigMap
  - path: argocd-rbac-cm.yaml   ← overlays the argocd-rbac-cm ConfigMap
```

**`argocd-install/base/argocd-cm.yaml`** — two key settings:
```yaml
data:
  application.resourceTrackingMethod: annotation
  # ↑ ArgoCD uses annotations (not labels) to track owned resources.
  #   Prevents conflicts with Helm's own labels on managed resources.

  resource.customizations.ignoreDifferences.apps_Deployment: |
    jqPathExpressions:
    - '.spec.replicas'
  # ↑ Ignores replica count drift on Deployments.
  #   Prevents sync loops when HPA scales pods — ArgoCD won't try to revert.
```

**`argocd-install/clusters/management/kustomization.yaml`:**
```yaml
resources:
  - ../../base             ← inherits everything from base
  - root-app-of-apps.yaml  ← adds the bootstrap Application
patches:
  - path: argocd-cm.yaml   ← overrides URL only
```

The kustomize merge works as: `base argocd-cm` is applied first, then `clusters/management/argocd-cm` is merged on top — only the fields present in the patch file are overwritten, everything else is kept.

**Bootstrap command (runs once):**
```bash
kubectl apply -k argocd-install/clusters/management/
```

This creates ~60 K8s resources including all ArgoCD components **and** the `root-app-of-apps` Application CR in one shot.

---

## 5. Layer 1 — Root App-of-Apps Bootstrap

**`argocd-install/clusters/management/root-app-of-apps.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app-of-apps
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: '-1'
spec:
  destination:
    name: in-cluster        ← deploys INTO ArgoCD's own cluster
    namespace: argocd       ← Application CRs land in the argocd namespace
  source:
    path: app-of-apps/clusters/management   ← watches this Git path
    repoURL: https://github.com/deegupt4/k8s-manage-app
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      prune: true       ← removes Applications if their yaml is deleted from Git
      selfHeal: true    ← reverts manual changes in the cluster back to Git state
```

**What ArgoCD does with this Application:**

1. Clones `https://github.com/deegupt4/k8s-manage-app` at branch `main`
2. Runs `kustomize build app-of-apps/clusters/management/`
3. The output is 4 Application CRs (one per workload cluster)
4. Applies those CRs to `in-cluster` in the `argocd` namespace
5. Watches for Git changes — any push to `app-of-apps/clusters/management/` triggers re-sync

**`app-of-apps/clusters/management/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd         ← all resources created in the argocd namespace
resources:
  - apps-hou-sbx-01.yaml
  - apps-kty-dev-01.yaml
  - apps-kty-prd-01.yaml
  - apps-kty-prd-02.yaml
```

Result: 4 Application objects now live in ArgoCD. Each one watches a different `apps/clusters/{cluster}/` directory.

---

## 6. Layer 2 — App-of-Apps (Per Cluster)

Each `apps-{cluster}.yaml` is an Application CR that points to `apps/clusters/{cluster}/`:

**`app-of-apps/clusters/management/apps-kty-prd-01.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kty-prd-01.apps
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: '-1'   ← runs before default wave 0
spec:
  destination:
    name: in-cluster     ← these Application CRs also land on the management cluster
    namespace: argocd    ← NOT on kty-prd-01 — only the workloads go there
  source:
    path: apps/clusters/kty-prd-01
    repoURL: https://github.com/deegupt4/k8s-manage-app
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Key point — destination is always `in-cluster`:**
The `apps-{cluster}` Application creates more Application CRs on the *management* cluster. It does NOT deploy workloads directly. The workload Applications it creates then target the actual workload cluster.

**`apps/clusters/kty-prd-01/kustomization.yaml`:**
```yaml
resources:
  - webapp/     ← includes apps/clusters/kty-prd-01/webapp/kustomization.yaml
  - cache/      ← includes apps/clusters/kty-prd-01/cache/kustomization.yaml
  - database/   ← includes apps/clusters/kty-prd-01/database/kustomization.yaml
```

Each sub-kustomization just lists one file:
```yaml
# apps/clusters/kty-prd-01/webapp/kustomization.yaml
resources:
  - webapp.yaml
```

So `kustomize build apps/clusters/kty-prd-01/` outputs 3 resources: `kty-prd-01.webapp`, `kty-prd-01.cache` (ApplicationSet), `kty-prd-01.database`.

---

## 7. Layer 3 — Application & ApplicationSet CRs

### Pattern A — Direct Application with Multi-Source Helm

Used by: `webapp`, `database`, `cache` on dev/sbx clusters.

**`apps/clusters/kty-dev-01/webapp/webapp.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kty-dev-01.webapp
  namespace: argocd
  labels:
    environment: dev
    cluster: kty-dev-01
spec:
  project: webapp
  sources:
    - repoURL: ghcr.io/stefanprodan/charts   ← OCI Helm registry
      chart: podinfo
      targetRevision: 6.7.1
      helm:
        releaseName: webapp
        valueFiles:
          - $values/manifests/clusters/kty-dev-01/values.yaml
          - $values/manifests/webapp/values.yaml
          - $values/manifests/webapp/clusters/kty-dev-01/values.yaml
          #  ↑ $values is resolved from the second source below
    - repoURL: https://github.com/deegupt4/k8s-manage-app
      targetRevision: main
      ref: values   ← this source has no path — it's a named reference only
      #               ArgoCD makes it available as $values in valueFiles above
  destination:
    name: kty-dev-01       ← the registered workload cluster name
    namespace: webapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true   ← ArgoCD creates the namespace on first sync
```

**How multi-source works:**
- Source 1 = the Helm chart (no values, just the chart)
- Source 2 = the Git repo with `ref: values` — this makes the entire repo root available as `$values`
- The `$values` variable in `valueFiles` expands to the root of source 2
- ArgoCD merges all 3 value files and passes them to `helm template`

### Pattern B — ApplicationSet with Git File Generator

Used by: `cache` on `kty-prd-01` and `kty-prd-02`.

**`apps/clusters/kty-prd-01/cache/cache.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kty-prd-01.cache
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/deegupt4/k8s-manage-app
        revision: main
        files:
          - path: manifests/cache/clusters/kty-prd-01/*/config.yaml
          #  ↑ glob — matches any config.yaml one level deep
          #    currently matches:
          #      manifests/cache/clusters/kty-prd-01/primary/config.yaml
          #      manifests/cache/clusters/kty-prd-01/replica/config.yaml
  template:
    metadata:
      name: 'kty-prd-01.cache-{{shard.name}}'
      #              ↑ variable from config.yaml content
    spec:
      project: cache
      sources:
        - repoURL: https://github.com/deegupt4/k8s-manage-app
          targetRevision: main
          ref: repo           ← named ref, available as $repo below
        - repoURL: https://charts.bitnami.com/bitnami
          chart: redis
          targetRevision: 19.6.4
          helm:
            releaseName: 'cache-{{shard.name}}'
            valueFiles:
              - $repo/manifests/cache/values.yaml
              - $repo/manifests/cache/clusters/kty-prd-01/values.yaml
              - $repo/manifests/cache/clusters/kty-prd-01/{{shard.name}}/values.yaml
              #                                             ↑ resolves to primary or replica
      destination:
        name: kty-prd-01
        namespace: cache
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**`manifests/cache/clusters/kty-prd-01/primary/config.yaml`:**
```yaml
shard:
  name: primary
  role: master
```

**How the generator works:**
1. ArgoCD scans Git for files matching `manifests/cache/clusters/kty-prd-01/*/config.yaml`
2. Each matching file is parsed as YAML — its fields become template variables
3. The template is rendered once per file → 2 Applications are generated:
   - `kty-prd-01.cache-primary` (using primary/values.yaml)
   - `kty-prd-01.cache-replica` (using replica/values.yaml)
4. If you add `manifests/cache/clusters/kty-prd-01/shard3/config.yaml`, a 3rd Application is automatically created on next sync — no changes to the ApplicationSet needed

---

## 8. Layer 4 — Helm Values Merge

Every Application specifies 3 value files. They are merged in order — later files override earlier ones.

**Merge order for `kty-prd-01.webapp`:**

```
FILE 1: manifests/clusters/kty-prd-01/values.yaml
─────────────────────────────────────────────────
global:
  cluster: kty-prd-01
  environment: prod
  imagePullSecrets:
    - name: registry-credentials

FILE 2: manifests/webapp/values.yaml
─────────────────────────────────────────────────
replicaCount: 1
ui:
  color: "#3d6fb5"
  message: "GitOps Demo"
service:
  type: ClusterIP
  port: 9898
resources:
  requests: { cpu: 50m, memory: 64Mi }
  limits:   { cpu: 200m, memory: 128Mi }
livenessProbe: { httpGet.path: /healthz, ... }
readinessProbe: { httpGet.path: /readyz, ... }

FILE 3: manifests/webapp/clusters/kty-prd-01/values.yaml
─────────────────────────────────────────────────
replicaCount: 3             ← overrides FILE 2's replicaCount: 1
ui:
  color: "#f44336"          ← overrides FILE 2's color
  message: "GitOps Demo — Production (kty-prd-01)"
ingress:
  enabled: true             ← new key, not in FILE 2
  host: webapp.prd-01.demo.example.com

FINAL MERGED VALUES passed to helm template:
─────────────────────────────────────────────────
global.cluster: kty-prd-01
global.environment: prod
global.imagePullSecrets: [registry-credentials]
replicaCount: 3
ui.color: "#f44336"
ui.message: "GitOps Demo — Production (kty-prd-01)"
service.type: ClusterIP
service.port: 9898
resources: (from FILE 2, unchanged)
ingress.enabled: true
ingress.host: webapp.prd-01.demo.example.com
livenessProbe: (from FILE 2, unchanged)
readinessProbe: (from FILE 2, unchanged)
```

**Why this 3-level structure:**

| File | Scope | Who edits it |
|------|-------|-------------|
| `manifests/clusters/{cluster}/values.yaml` | Cross-app cluster metadata | Platform team |
| `manifests/{app}/values.yaml` | App defaults (all envs) | App team |
| `manifests/{app}/clusters/{cluster}/values.yaml` | Env-specific overrides | App team |

---

## 9. ApplicationSet — Git File Generator Pattern

### How ArgoCD discovers config.yaml files

```
manifests/cache/clusters/kty-prd-01/
├── values.yaml                 ← cluster-wide cache values (not discovered, manually referenced)
├── primary/
│   ├── config.yaml             ← DISCOVERED — creates kty-prd-01.cache-primary
│   └── values.yaml             ← referenced via {{shard.name}} in ApplicationSet template
└── replica/
    ├── config.yaml             ← DISCOVERED — creates kty-prd-01.cache-replica
    └── values.yaml
```

### Resulting Applications

| Generated App | config.yaml source | Values used |
|--------------|-------------------|-------------|
| `kty-prd-01.cache-primary` | primary/config.yaml | values.yaml → kty-prd-01/values.yaml → primary/values.yaml |
| `kty-prd-01.cache-replica` | replica/config.yaml | values.yaml → kty-prd-01/values.yaml → replica/values.yaml |

### Adding a new shard

To add a third shard (e.g. `sentinel`):

```bash
mkdir -p manifests/cache/clusters/kty-prd-01/sentinel
cat > manifests/cache/clusters/kty-prd-01/sentinel/config.yaml << EOF
shard:
  name: sentinel
  role: sentinel
EOF
cat > manifests/cache/clusters/kty-prd-01/sentinel/values.yaml << EOF
architecture: sentinel
sentinel:
  enabled: true
  quorum: 2
EOF
```

On next Git sync → ArgoCD automatically creates `kty-prd-01.cache-sentinel`. No changes to the ApplicationSet YAML needed.

---

## 10. ArgoCD Projects (RBAC)

AppProjects scope what each Application can access. Every Application references a project.

**`argocd-projects/base/webapp.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: webapp
  namespace: argocd
spec:
  sourceRepos:
    - '*'                        ← can pull from any Helm registry or Git repo
  destinations:
    - namespace: 'webapp*'       ← Applications in this project can ONLY deploy to webapp* namespaces
      server: '*'                ← on any cluster
  roles:
    - name: sync-role
      policies:
        - p, proj:webapp:sync-role, applications, sync, webapp/*, allow
        - p, proj:webapp:sync-role, applications, get, webapp/*, allow
      groups:
        - frontend-team          ← maps to an SSO group (GitHub org team, Okta group, etc.)
```

**How projects are deployed:**

```
argocd-projects/
├── base/
│   ├── kustomization.yaml  ← lists webapp.yaml, cache.yaml, database.yaml
│   ├── webapp.yaml
│   ├── cache.yaml
│   └── database.yaml
└── clusters/management/
    └── kustomization.yaml  ← resources: ../../base
```

The `argocd-projects/clusters/management/` directory is deployed by the `mgmt.infra` Application (if configured) or can be applied manually:
```bash
kubectl apply -k argocd-projects/clusters/management/
```

**If an Application references a project that doesn't exist**, ArgoCD will show it as `Unknown` and refuse to sync.

---

## 11. Cluster Registration

Before ArgoCD can deploy to a workload cluster, that cluster must be registered. This requires:

### Step 1 — Apply service account to workload cluster

`argocd-service-accounts/sa.yaml` creates on the workload cluster:

```yaml
# What gets created on the workload cluster (e.g. kty-dev-01):
Namespace: argocd
ServiceAccount: argocd-manager (in argocd namespace)
Secret: argocd-manager-token (type: kubernetes.io/service-account-token)
  ↑ K8s auto-populates .data.token with a JWT for the SA
ClusterRole: argocd-manager-role
  rules: [apiGroups: '*', resources: '*', verbs: '*']  ← full cluster admin
ClusterRoleBinding: argocd-manager-role-binding
  ↑ binds ClusterRole to argocd-manager ServiceAccount
```

```bash
kubectx kty-dev-01
kubectl apply -f argocd-service-accounts/sa.yaml
```

### Step 2 — Extract the token

```bash
TOKEN=$(kubectl -n argocd get secret argocd-manager-token \
  -o jsonpath='{.data.token}' | base64 -d)
CA=$(kubectl -n argocd get secret argocd-manager-token \
  -o jsonpath='{.data.ca\.crt}')
SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
```

### Step 3 — Register with ArgoCD

```bash
kubectx management
argocd cluster add kty-dev-01 --name kty-dev-01
# ↑ argocd CLI reads the kty-dev-01 kubeconfig context and creates a Secret
#   in the argocd namespace on the management cluster
```

**What argocd cluster add creates:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-kty-dev-01-<hash>
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster   ← ArgoCD watches Secrets with this label
data:
  name: kty-dev-01
  server: https://<api-server-url>
  config: |
    { "bearerToken": "<SA token>", "tlsClientConfig": { "caData": "<ca.crt>" } }
```

ArgoCD watches Secrets with label `argocd.argoproj.io/secret-type: cluster` — this is how it knows which clusters it manages.

---

## 12. How to Add a New Cluster

Example: adding `kty-stg-01` (staging cluster).

### Files to create

**1. Register the cluster (Steps 1-3 from section 11)**

**2. Create `app-of-apps/clusters/management/apps-kty-stg-01.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kty-stg-01.apps
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: '-1'
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    name: in-cluster
    namespace: argocd
  source:
    path: apps/clusters/kty-stg-01
    repoURL: https://github.com/deegupt4/k8s-manage-app
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**3. Add to `app-of-apps/clusters/management/kustomization.yaml`:**
```yaml
resources:
  - apps-hou-sbx-01.yaml
  - apps-kty-dev-01.yaml
  - apps-kty-prd-01.yaml
  - apps-kty-prd-02.yaml
  - apps-kty-stg-01.yaml    ← add this line
```

**4. Create `apps/clusters/kty-stg-01/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - webapp/
  - cache/
  - database/
```

**5. Create Application CRs** (copy from an existing cluster and update `name`, `cluster`, `environment`, `destination.name`):
```
apps/clusters/kty-stg-01/webapp/kustomization.yaml
apps/clusters/kty-stg-01/webapp/webapp.yaml        ← change: name, destination, environment label
apps/clusters/kty-stg-01/cache/kustomization.yaml
apps/clusters/kty-stg-01/cache/cache.yaml          ← change: name, destination
apps/clusters/kty-stg-01/database/kustomization.yaml
apps/clusters/kty-stg-01/database/database.yaml    ← change: name, destination
```

**6. Create shared cluster values `manifests/clusters/kty-stg-01/values.yaml`:**
```yaml
global:
  cluster: kty-stg-01
  environment: staging
  imagePullSecrets:
    - name: registry-credentials
```

**7. Create per-app cluster overrides:**
```
manifests/webapp/clusters/kty-stg-01/values.yaml
manifests/cache/clusters/kty-stg-01/values.yaml
manifests/database/clusters/kty-stg-01/values.yaml
```

**8. PR → merge → ArgoCD auto-syncs** — the `root-app-of-apps` detects the new `apps-kty-stg-01.yaml`, creates `kty-stg-01.apps`, which then creates all workload Applications.

### Summary of files changed/created

```
+ app-of-apps/clusters/management/apps-kty-stg-01.yaml         (new)
~ app-of-apps/clusters/management/kustomization.yaml            (1 line added)
+ apps/clusters/kty-stg-01/kustomization.yaml                   (new)
+ apps/clusters/kty-stg-01/webapp/{kustomization,webapp}.yaml   (new)
+ apps/clusters/kty-stg-01/cache/{kustomization,cache}.yaml     (new)
+ apps/clusters/kty-stg-01/database/{kustomization,database}.yaml (new)
+ manifests/clusters/kty-stg-01/values.yaml                     (new)
+ manifests/webapp/clusters/kty-stg-01/values.yaml              (new)
+ manifests/cache/clusters/kty-stg-01/values.yaml               (new)
+ manifests/database/clusters/kty-stg-01/values.yaml            (new)
```

Total: **10 new files, 1 line edit**.

---

## 13. How to Add a New App

Example: adding `messaging` (RabbitMQ) to all clusters.

### Files to create

**1. Create the Application CR for each cluster:**

`apps/clusters/kty-dev-01/messaging/messaging.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kty-dev-01.messaging
  namespace: argocd
  labels:
    app.kubernetes.io/name: messaging
    environment: dev
    cluster: kty-dev-01
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: messaging
  sources:
    - repoURL: https://charts.bitnami.com/bitnami
      chart: rabbitmq
      targetRevision: 14.6.6
      helm:
        releaseName: messaging
        valueFiles:
          - $values/manifests/clusters/kty-dev-01/values.yaml
          - $values/manifests/messaging/values.yaml
          - $values/manifests/messaging/clusters/kty-dev-01/values.yaml
    - repoURL: https://github.com/deegupt4/k8s-manage-app
      targetRevision: main
      ref: values
  destination:
    name: kty-dev-01
    namespace: messaging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**2. Add to cluster kustomization `apps/clusters/kty-dev-01/kustomization.yaml`:**
```yaml
resources:
  - webapp/
  - cache/
  - database/
  - messaging/    ← add this
```

Repeat for each cluster that should get the app.

**3. Create global values `manifests/messaging/values.yaml`:**
```yaml
replicaCount: 1
auth:
  username: user
  password: ""    # set via ExternalSecret or existingSecret in prod
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
persistence:
  enabled: false
```

**4. Create per-cluster values `manifests/messaging/clusters/kty-dev-01/values.yaml`:**
```yaml
persistence:
  enabled: false
replicaCount: 1
```

**5. Create the AppProject `argocd-projects/base/messaging.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: messaging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: RabbitMQ messaging
  sourceRepos:
    - '*'
  destinations:
    - namespace: 'messaging*'
      server: '*'
  roles:
    - name: sync-role
      policies:
        - p, proj:messaging:sync-role, applications, sync, messaging/*, allow
        - p, proj:messaging:sync-role, applications, get, messaging/*, allow
      groups:
        - platform-team
```

**6. Add to `argocd-projects/base/kustomization.yaml`:**
```yaml
resources:
  - webapp.yaml
  - cache.yaml
  - database.yaml
  - messaging.yaml    ← add this
```

**7. Apply the updated AppProject (or let ArgoCD sync it if you have a project-deploying app):**
```bash
kubectl apply -k argocd-projects/clusters/management/
```

### Summary of files changed/created

```
+ apps/clusters/{each-cluster}/messaging/kustomization.yaml    (new, per cluster)
+ apps/clusters/{each-cluster}/messaging/messaging.yaml        (new, per cluster)
~ apps/clusters/{each-cluster}/kustomization.yaml              (1 line added, per cluster)
+ manifests/messaging/values.yaml                              (new)
+ manifests/messaging/clusters/{each-cluster}/values.yaml      (new, per cluster)
+ argocd-projects/base/messaging.yaml                          (new)
~ argocd-projects/base/kustomization.yaml                      (1 line added)
```

---

## 14. Sync Waves & Ordering

Sync waves control the order ArgoCD applies resources within a single sync operation.

| Wave | Resources | Why |
|------|-----------|-----|
| `-1` | `apps-{cluster}` Applications in app-of-apps | Must create child Application CRs before wave 0 tries to use them |
| `-1` | `database` Applications | Database must be ready before webapp/cache start |
| `0` (default) | `webapp`, `cache` Applications | Deploy after infra is ready |

**How waves work:**
```
ArgoCD sync start
  → apply all wave -1 resources → wait for healthy
  → apply all wave 0 resources  → wait for healthy
  → apply all wave +1 resources → etc.
```

Set via annotation:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: '-1'
```

---

## 15. Environment Differences

| Setting | hou-sbx-01 (sbx) | kty-dev-01 (dev) | kty-prd-01/02 (prod) |
|---------|-----------------|-----------------|----------------------|
| **webapp replicas** | 1 | 1 | 3 |
| **webapp UI color** | purple `#9c27b0` | green `#4caf50` | red `#f44336` |
| **webapp ingress** | no | no | yes |
| **cache pattern** | Application (direct) | Application (direct) | ApplicationSet (primary + replica) |
| **cache persistence** | disabled | disabled | 8Gi per shard |
| **cache auth** | disabled | disabled | existingSecret |
| **database disk** | 5Gi | 5Gi | 50Gi |
| **database read replicas** | 0 | 0 | 1 |
| **database prune** | `false` | `false` | `false` |
| **database credentials** | hardcoded (demo) | hardcoded (demo) | existingSecret |
| **imagePullSecrets** | none | none | `registry-credentials` |
| **ArgoCD notifications** | no | no | Slack on fail/degraded |

---

## Quick Reference

### Bootstrap a new ArgoCD instance on management cluster
```bash
kubectx management
kubectl create namespace argocd
kubectl apply -k argocd-install/clusters/management/
```

### Add a workload cluster
```bash
kubectx <new-cluster>
kubectl apply -f argocd-service-accounts/sa.yaml
kubectx management
argocd cluster add <new-cluster> --name <new-cluster>
# then add files as described in section 12
```

### Force resync an app
```bash
argocd app sync kty-prd-01.webapp
```

### Check why an app is out of sync
```bash
argocd app diff kty-prd-01.webapp
argocd app get kty-prd-01.webapp --show-operation
```

### See all generated apps from a cache ApplicationSet
```bash
argocd app list | grep cache
# kty-prd-01.cache-primary
# kty-prd-01.cache-replica
# kty-prd-02.cache-primary
# kty-prd-02.cache-replica
```
