# k3s-apps

GitOps-managed workloads for the **valhalla** k3s cluster (and any future
cluster wired to this repo). The companion to
[`k8s-provisioning`](https://github.com/borntoeatt/k8s-provisioning), which
handles the platform itself (Terraform/Ansible, Argo CD, MetalLB, Longhorn).

## How it works

A "root" Argo CD `Application` running in the cluster watches this repo's
`apps/` directory. Every YAML in `apps/` is itself an Argo CD `Application` —
the **app-of-apps** pattern. Each child `Application` points at where its
manifests actually live, which can be one of three places (see below).

```
This repo                                    Cluster
─────────                                    ───────
apps/*.yaml ─────────────────► apps-root  ───► spawns one Application per file
                                                  │
                                                  ├─► (a) external Helm chart
                                                  ├─► (b) external git repo (app's own)
                                                  └─► (c) this repo's manifests/<name>/
```

## Three patterns for adding an app

### (a) External Helm chart

The chart is published to a Helm registry (Bitnami, prometheus-community,
Jetstack, etc). The Application points at the registry; the values are inline
in the same file. No `manifests/<name>/` folder in this repo.

Examples already deployed this way: `cert-manager`, `kube-prometheus-stack`,
`sealed-secrets`.

```yaml
# apps/<name>.yaml
spec:
  source:
    repoURL: https://charts.jetstack.io       # the Helm repo
    chart: cert-manager
    targetRevision: v1.16.2
    helm:
      values: |
        crds:
          enabled: true
        nodeSelector:
          node-role.kubernetes.io/worker: "true"
```

### (b) External git repo (app has its own home)

The app's Kubernetes manifests live in **the app's own repo**, usually
alongside its Dockerfile and CI. This repo only has a pointer. Best pattern
for apps with their own dev cycle (CI auto-bumps image tags, etc).

Examples already deployed this way: `beleganski-jaas`.

```yaml
# apps/<name>.yaml — just a pointer, no manifests/ folder needed here
spec:
  source:
    repoURL: https://github.com/borntoeatt/<app-name>.git
    targetRevision: main
    path: k8s                # whichever folder in that repo has the manifests
  destination:
    namespace: <app-name>    # always set this explicitly
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Trap to avoid**: don't keep an Argo CD `Application` CR inside the synced
manifest path. The Application would then manage and overwrite *itself* on
each sync — the destination namespace flips back to whatever's in the file,
not what's in `apps-root`. If your app repo has an `argocd-app.yaml`, move it
to a sibling folder (e.g. `argocd/`) so the sync path only contains real
workload manifests.

### (c) In-repo manifests (this repo's `manifests/<name>/`)

For workloads that don't have a natural home repo of their own — typically
small platform glue or one-off tooling. Manifests live in
`manifests/<name>/`; Application points at `path: manifests/<name>` on this
repo itself.

Examples already deployed this way: `cert-manager-issuers`, `homepage`,
`longhorn-storageclasses`, `uptime-kuma`, `vaultwarden`.

```yaml
# apps/<name>.yaml
spec:
  source:
    repoURL: https://github.com/borntoeatt/k3s-apps.git
    targetRevision: main
    path: manifests/<name>
```

## Layout

```
apps/        Argo CD Application CRs, one per workload (every pattern)
manifests/   Plain Kubernetes manifests, one folder per workload — only for
             pattern (c). Patterns (a) and (b) have no folder here.
```

## Conventions

### DNS

- **Internal apps** use `*.valhalla.lan` hostnames. The homelab Technitium
  DNS server has a wildcard `*.valhalla.lan` A record → `192.168.0.180`
  (Traefik's MetalLB LoadBalancer IP). Every device that uses Technitium
  resolves automatically. No `/etc/hosts` editing.
- **Public apps** (anything proxied through Cloudflare to your real domain)
  use the real domain in the Ingress (`alex-jaas.dporkov.tech`) and rely on
  Cloudflare → NPM → Traefik routing. See the "NPM forwarding" subsection
  below for the only sane way to wire NPM up.

### TLS

cert-manager runs in the cluster against an internal CA. New ingresses for
**internal** apps should annotate:

```yaml
annotations:
  traefik.ingress.kubernetes.io/router.entrypoints: websecure
  cert-manager.io/cluster-issuer: valhalla-ca-issuer
spec:
  tls:
    - hosts: [<your-hostname>.valhalla.lan]
      secretName: <name>-tls
```

cert-manager auto-provisions the cert. Import the `valhalla-ca` root cert
into client trust stores once (Mac Keychain, browser store, etc) and every
internal app gets a green lock with no warnings.

**Public apps** that terminate TLS at Cloudflare don't need this — keep
their ingress on the `web` entrypoint and let Cloudflare handle HTTPS.

### Persistent storage

Two StorageClasses available:

| StorageClass | Replicas | Use for |
|---|---|---|
| `longhorn` (default) | 3 | Anything where data loss matters — Vaultwarden, Gitea, app DBs |
| `longhorn-1r` | 1 | Rebuildable data — Prometheus TSDB, search index caches, build artifacts |

Single-replica saves 3x disk at the cost of "if the one node holding this
replica dies, this PVC's data is lost." Fine for metrics, never for
secrets/user data.

**Single-replica RWO** stateful apps need `strategy.type: Recreate` on the
Deployment — RWO Longhorn volumes can't be attached to two pods at once, so
RollingUpdate deadlocks on the second pod's volume mount.

### Homepage dashboard discovery

Homepage (`gethomepage.dev`) at https://home.valhalla.lan/ auto-discovers
services from Ingress annotations. Every new app should add:

```yaml
# In the app's Ingress metadata.annotations:
gethomepage.dev/enabled: "true"
gethomepage.dev/name: "<display name>"
gethomepage.dev/description: "<short subtitle>"
gethomepage.dev/group: "<Platform|Apps|Monitoring|Security>"
gethomepage.dev/icon: "<icon-name>"           # see github.com/walkxcode/dashboard-icons
gethomepage.dev/href: "<URL users click>"     # for public apps via Cloudflare, set the real https URL
gethomepage.dev/siteMonitor: "http://<svc>.<ns>.svc.cluster.local"  # in-cluster health check
```

And the Deployment **must** carry `app.kubernetes.io/name: <ingress-name>`
on **both** `metadata.labels` and `spec.template.metadata.labels`:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
spec:
  selector:
    matchLabels:
      app: my-app                     # leave the existing selector unchanged (immutable)
  template:
    metadata:
      labels:
        app: my-app
        app.kubernetes.io/name: my-app  # <-- THIS is what Homepage looks for
```

Without that label the tile shows "NOT FOUND / Offline" even when siteMonitor
returns 200 — Homepage's k8s integration finds no pods to count.

The `siteMonitor` should point at the **in-cluster Service URL**, not the
public URL. Hitting the public URL from inside the cluster hairpins through
Cloudflare → NPM → back into the cluster, and any one of those hops glitches
turn the tile red even when the app is fine.

### NPM forwarding rule

For every public app, the Nginx Proxy Manager entry should forward to:

```
Forward Hostname/IP:  192.168.0.180
Forward Port:         80           (Traefik's web entrypoint)
Scheme:               http
```

Traefik reads the `Host:` header (NPM passes it through) and uses the
matching Ingress rule to route to the right Service. **Do not forward to a
NodePort** — NodePorts get reassigned on every redeploy and the proxy
silently breaks the next time a pod recycles.

## Current apps

| Name | Pattern | URL | Notes |
|---|---|---|---|
| `cert-manager` | (a) Helm | — | TLS automation |
| `cert-manager-issuers` | (c) in-repo | — | Self-signed `valhalla-ca` root + `valhalla-ca-issuer` ClusterIssuer |
| `longhorn-storageclasses` | (c) in-repo | — | `longhorn-1r` (1-replica) StorageClass for rebuildable data |
| `sealed-secrets` | (a) Helm | — | Encrypt secrets at rest in this repo (controller: `sealed-secrets-controller`) |
| `kube-prometheus-stack` | (a) Helm | https://grafana.valhalla.lan/ | Prometheus + Alertmanager + Grafana with prebuilt dashboards |
| `homepage` | (c) in-repo | https://home.valhalla.lan/ | Cluster dashboard, auto-discovers via Ingress annotations |
| `uptime-kuma` | (c) in-repo | http://uptime.valhalla.lan/ | Service uptime monitor, SQLite on Longhorn |
| `vaultwarden` | (c) in-repo | https://vault.valhalla.lan/ | Bitwarden-compatible password server, SQLite on Longhorn |
| `beleganski-jaas` | (b) external | https://alex-jaas.dporkov.tech/ | Public web app — manifests live in [borntoeatt/beleganski-jaas](https://github.com/borntoeatt/beleganski-jaas), CI bumps image tags |

## When apps-root won't pick up a change

```bash
# Force apps-root to re-pull from git
kubectl -n argocd patch app apps-root --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Verify a child Application picked up the new spec from apps-root
kubectl -n argocd get app <name> -o jsonpath='{.spec.destination.namespace}{"\n"}'
```

If a child app's spec stays "stuck" on an old value even after apps-root
syncs (most common after creating Applications via the Argo CD UI rather
than via git), delete the child Application and let apps-root recreate it
from git:

```bash
kubectl -n argocd delete application <name>
# apps-root recreates within ~3 min, or force the refresh above
```
