# k3s-apps

GitOps-managed workloads for the **valhalla** k3s cluster (and any future
cluster wired to this repo). The companion to
[`k8s-provisioning`](https://github.com/borntoeatt/k8s-provisioning), which
handles the platform itself (Terraform/Ansible, Argo CD, MetalLB, Longhorn).

## How it works

A "root" Argo CD `Application` running in the cluster watches this repo's
`apps/` directory. Every YAML in `apps/` is itself an Argo CD `Application` —
the **app-of-apps** pattern. Each child Application points at a folder under
`manifests/` containing the actual Kubernetes resources.

```
This repo                                    Cluster
─────────                                    ───────
apps/uptime-kuma.yaml ────────►  Argo CD ◄──── root Application (watches apps/)
manifests/uptime-kuma/* ───────► Argo CD ◄──── uptime-kuma Application
                                              │
                                              └─► Deployment, Service, PVC, Ingress
```

## Adding a new app

1. Create `manifests/<name>/` with plain Kubernetes manifests
   (Deployment, Service, PVC, Ingress, etc).
2. Create `apps/<name>.yaml` — an Argo CD `Application` pointing at
   `manifests/<name>` on this repo.
3. Commit and push. Argo CD picks it up within a few minutes (or
   immediately if you click "Refresh" in the UI).

## Layout

```
apps/        Argo CD Application CRs, one per workload
manifests/   Plain Kubernetes manifests, one folder per workload
```

## Conventions

- **Hostnames** use the `*.valhalla.lan` pattern. Add an `/etc/hosts` entry
  on each client machine pointing the name at `192.168.0.180` (Traefik's
  MetalLB-assigned IP), or set up a wildcard `*.valhalla.lan` A record on
  your homelab DNS server.
- **TLS** is handled by `cert-manager` against an internal CA
  (`valhalla-ca-issuer`). New ingresses should annotate
  `cert-manager.io/cluster-issuer: valhalla-ca-issuer` and add a `tls`
  block — cert-manager will provision the cert automatically. Import the
  `valhalla-ca` root cert into your client trust store once and every app
  gets a green-lock cert.
- **Persistent storage** uses the `longhorn` StorageClass (default on
  this cluster). PVCs without an explicit `storageClassName` get Longhorn
  automatically.
- **Single-replica stateful apps** use `strategy.type: Recreate` because
  RWO Longhorn volumes can't be attached to two pods simultaneously.

## Current apps

| Name | URL | Notes |
|---|---|---|
| `cert-manager` | — | TLS automation for the cluster |
| `cert-manager-issuers` | — | Self-signed `valhalla-ca` root + `valhalla-ca-issuer` ClusterIssuer |
| `sealed-secrets` | — | Encrypt secrets at rest in this git repo (controller name: `sealed-secrets-controller`) |
| `kube-prometheus-stack` | https://grafana.valhalla.lan/ | Prometheus + Alertmanager + Grafana with prebuilt cluster dashboards |
| `uptime-kuma` | http://uptime.valhalla.lan/ | Service uptime monitor, SQLite on Longhorn |
| `vaultwarden` | https://vault.valhalla.lan/ | Bitwarden-compatible password server, SQLite on Longhorn, internal CA cert |
