# ArgoCD Multi-Cluster GitOps — Design Notes

Notes on the hub-spoke ArgoCD setup I'm building across 5 k0s clusters.
The hub runs ArgoCD + Harbor. Four spoke clusters cover sandbox, dev, and prod.
The chart lives in both Git and Harbor OCI — useful for testing dual-source setups.

---

## Cluster Topology

| Name        | IP             | Role                          |
|-------------|----------------|-------------------------------|
| management  | 10.10.141.188  | ArgoCD + Harbor (hub)         |
| hou-sbx-01  | 10.10.158.86   | Sandbox spoke                 |
| kty-dev-01  | 10.10.146.195  | Dev spoke                     |
| kty-prd-01  | 10.10.88.136   | Production spoke 1            |
| kty-prd-02  | 10.10.85.109   | Production spoke 2            |

---

## Repository Layout

I'm splitting into two repos to keep concerns separate:

### `gitops-charts` — Helm charts (also pushed to Harbor OCI)

```
gitops-charts/
├── Makefile                          # make push-to-harbor, lint, package
├── scripts/
│   └── push-to-harbor.sh             # helm push + crane image mirror
└── charts/
    └── podinfo/                      # using stefanprodan/podinfo as demo app
        ├── Chart.yaml                # name: podinfo, version: 1.0.0, appVersion: 6.7.0
        ├── values.yaml               # base defaults inherited by all clusters
        ├── values-sandbox.yaml       # grey UI, 1 replica, tiny resources, HPA off
        ├── values-development.yaml   # blue UI, 1 replica, small resources, HPA off
        ├── values-production.yaml    # green UI, 3 replicas, HPA on (3-10)
        └── templates/
            ├── _helpers.tpl
            ├── namespace.yaml        # sync-wave: -1
            ├── deployment.yaml
            ├── service.yaml
            ├── ingress.yaml
            ├── hpa.yaml              # gated by values flag
            ├── configmap.yaml        # ui.color and ui.message
            └── NOTES.txt
```

I chose podinfo because the web UI shows pod name, color, and message — you can open 4 browser
tabs and immediately see which cluster/env is running what. No database, no complexity.

**Per-environment visual differentiation:**

| Cluster     | UI Color | Message                        | Replicas | HPA   |
|-------------|----------|--------------------------------|----------|-------|
| hou-sbx-01  | grey     | SANDBOX — Experimental         | 1        | off   |
| kty-dev-01  | blue     | DEVELOPMENT                    | 1        | off   |
| kty-prd-01  | green    | PRODUCTION — kty-prd-01        | 3        | 3-10  |
| kty-prd-02  | green    | PRODUCTION — kty-prd-02        | 3        | 3-10  |
| kty-dev-01* | red      | FROM HARBOR OCI — kty-dev-01   | 1        | off   |

*Second deploy on dev, different namespace, sourced from Harbor OCI instead of Git

---

### `gitops-config` — ArgoCD configuration (the only repo ArgoCD watches)

```
gitops-config/
├── bootstrap/
│   ├── root-app.yaml                 # the only object I manually apply
│   └── cluster-secrets/              # secrets managed out-of-band, .gitkeep only
│       └── .gitkeep
│
├── argocd/                           # ArgoCD manages its own config via GitOps
│   ├── argocd-cm.yaml
│   └── argocd-rbac-cm.yaml
│
├── projects/
│   ├── project-sandbox.yaml          # sourceRepos: ["*"], open
│   ├── project-development.yaml      # git + harbor, kty-dev-01 only
│   └── project-production.yaml       # git + harbor only, prd clusters only
│
├── applicationsets/
│   ├── appset-podinfo-git.yaml       # cluster generator → all 4 spokes, source: git
│   └── appset-podinfo-harbor.yaml    # cluster generator → dev only, source: harbor OCI
│
├── apps/                             # app-of-apps children
│   ├── app-projects.yaml             # sync-wave: 0
│   ├── app-argocd-config.yaml        # sync-wave: 1
│   └── app-applicationsets.yaml      # sync-wave: 2
│
└── clusters/
    ├── hou-sbx-01/
    │   ├── cluster-metadata.yaml     # label/annotation reference for the cluster Secret
    │   └── values-override.yaml
    ├── kty-dev-01/
    │   ├── cluster-metadata.yaml
    │   └── values-override.yaml
    ├── kty-prd-01/
    │   ├── cluster-metadata.yaml
    │   └── values-override.yaml
    └── kty-prd-02/
        ├── cluster-metadata.yaml
        └── values-override.yaml
```

---

## App-of-Apps Cascade

One manual `kubectl apply` seeds the whole thing:

```
bootstrap/root-app.yaml   ← only manual apply
    │  (watches apps/)
    ├── app-projects.yaml        sync-wave: 0  → AppProjects
    ├── app-argocd-config.yaml   sync-wave: 1  → ArgoCD config
    └── app-applicationsets.yaml sync-wave: 2  → ApplicationSets
            │
            ├── appset-podinfo-git    → 1 Application per spoke (4 total)
            └── appset-podinfo-harbor → 1 Application on dev only
```

The sync waves ensure AppProjects exist before ApplicationSets try to reference them.

---

## ApplicationSet Cluster Generator

The cluster generator reads labels on ArgoCD cluster Secrets to drive templating:

| Cluster     | label: environment | label: region | annotation: values-file | annotation: ingress-suffix |
|-------------|-------------------|---------------|-------------------------|----------------------------|
| hou-sbx-01  | sandbox           | hou           | values-sandbox.yaml     | sbx.demo.local             |
| kty-dev-01  | development       | kty           | values-development.yaml | dev.demo.local             |
| kty-prd-01  | production        | kty           | values-production.yaml  | prd-01.demo.local          |
| kty-prd-02  | production        | kty           | values-production.yaml  | prd-02.demo.local          |

The ApplicationSet template injects cluster identity as Helm values:

```yaml
values: |
  clusterName: {{name}}
  ingress:
    host: podinfo.{{metadata.annotations.cluster.demo/ingress-host-suffix}}
```

This means I don't need separate Application YAMLs per cluster — the generator handles it.

---

## Harbor Setup

- Installed on management cluster, namespace `harbor`, NodePort 30080 (HTTP, no TLS for local lab)
- Harbor project `demo` (public):
  - `harbor.hub.local:30080/demo/images/podinfo:6.7.0` — container image
  - `harbor.hub.local:30080/demo/charts/podinfo:1.0.0` — OCI Helm chart
- All spokes pull container images from Harbor
- ArgoCD repo credential: `oci://harbor.hub.local:30080/demo/charts`

Useful commands:
```bash
# Push chart to Harbor OCI
helm package charts/podinfo
helm push podinfo-1.0.0.tgz oci://harbor.hub.local:30080/demo/charts

# Mirror image to Harbor
crane copy ghcr.io/stefanprodan/podinfo:6.7.0 harbor.hub.local:30080/demo/images/podinfo:6.7.0
```

---

## Bootstrap Sequence

```
Phase 0 — Tools on management VM
  install: helm, argocd CLI, crane
  /etc/hosts: harbor.hub.local + argocd.hub.local → 10.10.141.188

Phase 1 — k0s on management
  k0s install controller --single (local, not SSH)
  local-path-provisioner → default StorageClass (Harbor needs PVCs)
  merge context into ~/.kube/config

Phase 2 — Harbor
  helm install harbor harbor/harbor --set expose.type=nodePort --set expose.tls.enabled=false
  create "demo" project via Harbor API
  configure containerd insecure registry on all 4 spokes
  add harbor.hub.local to /etc/hosts on all spokes

Phase 3 — ArgoCD
  kubectl apply -f argocd install manifest
  expose argocd-server NodePort 32443
  argocd login + rotate initial password

Phase 4 — Register spoke clusters
  argocd cluster add <context> for each spoke
  kubectl patch cluster Secrets: add environment/region labels + ingress annotations

Phase 5 — Ingress on spokes
  helm install ingress-nginx on each spoke (NodePort 30080)
  /etc/hosts on local machine: map spoke IPs to demo hostnames

Phase 6 — Push chart + image to Harbor
  crane copy → harbor image
  helm package + helm push → harbor OCI chart

Phase 7 — ArgoCD repo connections
  argocd repo add gitops-charts (git)
  argocd repo add gitops-config (git)
  argocd repo add harbor OCI (--enable-oci --insecure-skip-server-verification)

Phase 8 — Bootstrap
  git push both repos
  kubectl apply -n argocd -f gitops-config/bootstrap/root-app.yaml
  watch the cascade in ArgoCD UI
```

---

## Things Left to Build

- [ ] `gitops-charts/charts/podinfo/` — Chart.yaml, all values files, templates
- [ ] `gitops-charts/scripts/push-to-harbor.sh` + Makefile
- [ ] `gitops-config/bootstrap/root-app.yaml`
- [ ] `gitops-config/apps/` — 3 Application YAMLs with sync waves
- [ ] `gitops-config/projects/` — 3 AppProject YAMLs
- [ ] `gitops-config/applicationsets/appset-podinfo-git.yaml`
- [ ] `gitops-config/applicationsets/appset-podinfo-harbor.yaml`
- [ ] `gitops-config/clusters/*/values-override.yaml` (4 files)
- [ ] k0s setup script updates for management VM Phase 1
