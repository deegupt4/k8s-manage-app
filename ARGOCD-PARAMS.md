# ArgoCD Parameter Reference

Personal reference for ArgoCD Application/ApplicationSet fields used in this repo.

---

## sync-wave

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

Controls ordering within a single sync operation. ArgoCD syncs in ascending wave order and
waits for all resources in wave N to be `Healthy` before starting wave N+1.

- Default wave is `0`
- Negative values run before `0` — use for Namespaces, CRDs, AppProjects
- Resources within the same wave sync in parallel

Common ordering pattern:
```
-1 → Namespaces, CRDs
 0 → AppProjects, RBAC
 1 → Secrets, ConfigMaps
 2 → Deployments, StatefulSets
 3 → Ingress, HPA
```

---

## finalizers

```yaml
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
```

Controls what happens when the Application CR itself is deleted.

| Finalizer | Behaviour |
|-----------|-----------|
| `resources-finalizer.argocd.argoproj.io` | Cascade-deletes all managed resources when Application is deleted |
| *(none)* | Deletes the Application object only; deployed resources remain (orphan) |

Use cascade for stateless apps. Omit for databases and anything with PVCs — a
`git revert` or accidental deletion should not wipe persistent data.

---

## syncPolicy

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### automated

Without `automated`, ArgoCD only detects drift — you must manually click Sync or run
`argocd app sync`. With it, ArgoCD syncs automatically on git changes.

| Field | Off behaviour | On behaviour |
|-------|--------------|-------------|
| `prune` | Resources removed from git stay running in the cluster | Removed resources are deleted from the cluster |
| `selfHeal` | Manual `kubectl edit` changes persist | ArgoCD reverts cluster state back to git within seconds |

### syncOptions

| Option | What it does |
|--------|-------------|
| `CreateNamespace=true` | Creates the destination namespace if it doesn't exist |
| `ServerSideApply=true` | Uses server-side apply. Required for large CRDs to avoid annotation size limits |
| `Replace=true` | Uses `kubectl replace` instead of `apply`. For resources with immutable fields |
| `ApplyOutOfSyncOnly=true` | Only applies resources that are actually out of sync — faster syncs on large apps |
| `PruneLast=true` | Runs pruning as the final step after all sync waves complete |
| `PrunePropagationPolicy=foreground` | Controls cascade delete propagation: `foreground`, `background`, or `orphan` |
| `Validate=false` | Skips `kubectl --validate`. Useful when CRDs aren't registered yet at apply time |
| `RespectIgnoreDifferences=true` | Applies `ignoreDifferences` rules during sync, not just during diff display |

### retry

Retries failed syncs with exponential backoff. Without this, a transient failure (webhook
timeout, API server blip) leaves the app stuck in `SyncFailed` until manual intervention.

```yaml
retry:
  limit: 5           # -1 = infinite
  backoff:
    duration: 5s     # initial wait
    factor: 2        # multiplier each attempt
    maxDuration: 3m  # cap
```

---

## ignoreDifferences

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    - group: ""
      kind: Secret
      managedFieldsManagers:
        - kube-controller-manager
```

Tells ArgoCD to ignore specific fields when computing drift. Without this, any controller
that mutates a field causes the app to show `OutOfSync` permanently.

| `jsonPointers` | `managedFieldsManagers` |
|----------------|------------------------|
| Match by JSON Pointer path | Match by field manager name (server-side apply) |
| e.g. `/spec/replicas` | e.g. `kube-controller-manager`, `cert-manager` |

Common cases:
- `/spec/replicas` — HPA manages replica count
- `/spec/template/spec/containers/0/image` — CI updates image tags directly
- `caBundle` fields — injected by admission webhook controllers
- Secrets injected by cert-manager or external-secrets

---

## project

```yaml
spec:
  project: production
```

References an AppProject which defines:
- Which source repos the app may use
- Which destination clusters and namespaces it may deploy to
- Which Kubernetes resource kinds it can manage
- RBAC: which roles can sync/manage it

Using `default` project means no restrictions. Named projects scope blast radius and
enforce governance — use them for anything beyond a personal lab.

See `argocd-projects/` in this repo for the AppProject definitions.

---

## destination

```yaml
spec:
  destination:
    name: kty-prd-01        # registered cluster name
    # server: https://...   # or raw API URL — don't use both
    namespace: webapp
```

- `name` — references the cluster by the name it was registered with in ArgoCD
- `server` — the raw Kubernetes API server URL
- `name: in-cluster` — special value for the cluster ArgoCD itself runs on (management cluster here)

---

## sources (multi-source)

```yaml
spec:
  sources:
    - repoURL: https://charts.bitnami.com/bitnami
      chart: redis
      targetRevision: 19.6.4
      helm:
        valueFiles:
          - $repo/manifests/redis/values.yaml
    - repoURL: https://github.com/deegupt4/k8s-manage-app
      targetRevision: main
      ref: repo        # pure reference — not deployed itself
```

Allows combining a Helm chart from one source with values files from a separate git repo.
The source with `ref:` is a named reference only — ArgoCD does not deploy it. Other sources
reference it with `$<ref-name>` in their `valueFiles` paths.

This is the core pattern in this repo: Helm chart from a registry, values from git.

---

## helm block

```yaml
helm:
  releaseName: my-app
  valueFiles:
    - values.yaml
    - $repo/path/to/cluster-values.yaml
  values: |
    replicaCount: 3
    image:
      tag: v1.2.3
  parameters:
    - name: image.tag
      value: v1.2.3
  ignoreMissingValueFiles: true
  skipCrds: false
```

| Field | Purpose |
|-------|---------|
| `releaseName` | Override Helm release name (default: Application name) |
| `valueFiles` | Values files to merge, left to right — last file wins on conflicts |
| `values` | Inline values block, merged last (highest priority) |
| `parameters` | Individual `--set` style overrides |
| `ignoreMissingValueFiles` | Don't fail if a values file path doesn't exist — useful for optional per-cluster overrides |
| `skipCrds` | Don't install CRDs bundled in the chart |

Values merge order (lowest → highest priority):
```
chart defaults → valueFiles[0] → valueFiles[1] → ... → values (inline) → parameters
```

---

## kustomize block

```yaml
kustomize:
  version: v4.5.7
  images:
    - my-app=registry/my-app:v1.2.3
  commonLabels:
    environment: production
  patches:
    - path: patch.yaml
```

Injects kustomize-level overrides without modifying the `kustomization.yaml` in git.
Useful for image tag overrides from CI without touching the source files.

---

## ApplicationSet-specific fields

```yaml
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: environment
              operator: In
              values: [sandbox]
        - matchExpressions:
            - key: environment
              operator: In
              values: [production]
```

| Field | Purpose |
|-------|---------|
| `preserveResourcesOnDeletion` | Orphans generated Applications when the ApplicationSet is deleted. Safe default for prod — prevents accidental mass-delete |
| `RollingSync` | Rolls out changes step by step across cluster groups. Deploy to sandbox first, wait for health, then production |

Without `RollingSync`, all generated Applications sync simultaneously — a bad values change
hits every cluster at once.

---

## Quick Decision Guide

| Situation | What to set |
|-----------|-------------|
| Stateless app, full GitOps | `automated: {prune: true, selfHeal: true}` + cascade finalizer |
| Database / stateful workload | `prune: false`, no cascade finalizer |
| HPA manages replica count | `ignoreDifferences` on `/spec/replicas` |
| Resources need ordered deploy | `sync-wave` annotations |
| Namespace doesn't pre-exist | `CreateNamespace=true` syncOption |
| Large CRDs (annotation overflow) | `ServerSideApply=true` |
| Helm chart from registry + git values | `sources` array with `ref:` |
| Protect prod from simultaneous rollout | ApplicationSet `RollingSync` strategy |
| Transient sync failures in flaky env | `retry` with backoff |
| Controller mutates fields constantly | `ignoreDifferences` with jsonPointers |
