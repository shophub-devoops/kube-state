# GitOps with Argo CD (optional, spec 5.3)

This directory turns `kube-state` into an [Argo CD](https://argo-cd.readthedocs.io/)
**app-of-apps**. Instead of running `helm install` by hand (the manual flow in the
[top-level README](../README.md)), Argo CD continuously reconciles the cluster to
match this git repo: change a pinned version or a `values.yaml`, merge to `main`,
and Argo CD rolls it out.

## How it maps onto the existing structure

The component overrides stay where they were — `clusters/local/<component>/values.yaml`
remains the **single source of truth**. Each Argo CD `Application` is a *multi-source*
app: it pulls the chart from its upstream repo and the values file from this repo
(`$values/clusters/local/.../values.yaml`). Nothing is duplicated.

```
argocd/
├── project.yaml          # AppProject "shophub" (allowed repos + destinations)
├── repositories.yaml     # registers the DockerHub OCI registry (anonymous)
├── root.yaml             # the app-of-apps Application
└── apps/                 # one Application per component
    ├── cnpg.yaml                   (wave 0)
    ├── mongodb-operator.yaml       (wave 0)
    ├── kube-prometheus-stack.yaml  (wave 0)
    ├── loki.yaml                   (wave 1)
    ├── tempo.yaml                  (wave 1)
    ├── shop-operator.yaml          (wave 1)
    └── shophub.yaml                (wave 2)
```

### Sync waves (ordering)

`argocd.argoproj.io/sync-wave` enforces dependency order so a chart never installs
before the CRDs it needs exist:

- **Wave 0** — operators that publish CRDs: CNPG, MongoDB, kube-prometheus-stack
  (ServiceMonitor / PrometheusRule / AlertmanagerConfig).
- **Wave 1** — `loki`, `tempo`, and `shop-operator` (installs the Shop/DiscordChannel/
  Wallet CRDs; needs the monitoring CRDs from wave 0 for its PrometheusRule).
- **Wave 2** — `shophub` (uses the Shop CRD and the ServiceMonitor CRD).

## Bootstrap

```bash
# 1. create the k3d cluster (same as the manual flow)
k3d cluster create --config clusters/local/cluster.yaml

# 2. install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. one-time config: the project + the OCI registry registration
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/repositories.yaml

# 4. hand the cluster over to GitOps
kubectl apply -f argocd/root.yaml
```

From here Argo CD installs and self-heals everything. Watch it converge:

```bash
kubectl -n argocd get applications -w
# or open the UI:
kubectl -n argocd port-forward svc/argocd-server 8081:443
# admin password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

## Notes

- The two **OCI** charts (`shop-operator`, `shophub`) are pulled from DockerHub via
  the repository registered in `repositories.yaml`. They are public, so no
  credentials are stored.
- `ServerSideApply=true` is set on every app — kube-prometheus-stack's CRDs are too
  large for the client-side apply annotation.
- The manual `helm install` flow in the parent README still works; this is an
  optional GitOps layer on top of the same pinned versions and values.
