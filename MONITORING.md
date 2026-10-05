# Monitoring: Prometheus + Grafana via Helm (namespace `monitoring`)

The `kube-prometheus-stack` chart brings Prometheus + Grafana + Alertmanager +
exporters (node-exporter, kube-state-metrics) in a single release.
Fit check, done live: each worker has ~775-975m CPU and ~1.4-1.8Gi memory
free, 25G disk free. The stack below asks ~600m CPU / ~1.3Gi mem + ~17Gi
disk, so it fits on either worker (no affinity needed; the CP taint keeps it
off the control plane anyway).

Storage, production-style: the cluster ships ZERO StorageClasses, so step 0
installs Rancher's `local-path-provisioner` (one YAML; volumes live on the
node's own disk and survive pod restarts AND full VM reboots — virtual disks
persist). That is what makes this a faithful prod simulation instead of
`emptyDir` confetti. Honest lab trade-off, worth knowing for interviews:
local-path volumes are node-local — if a pod ever reschedules to the other
worker, the old data stays behind. Real prod uses network storage
(EBS/Longhorn/Ceph) and, for long retention, remote-write
(Thanos/Cortex/Mimir).

Run on the laptop, with the Doppler exports done:

### Step 0: Storage backend (once per cluster rebuild)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml"
ssh "$K8S_CP_CONN" -- "kubectl -n local-path-storage rollout status deploy/local-path-provisioner && kubectl get storageclass"
# expect: local-path ... (default). The values below pin it explicitly anyway.
```

### Step 1: Values file on the CP (quoted 'EOF' heredoc: nothing expands locally)

```bash
# Lab mirrors prod knobs: 1-minute scrape, 1-day retention, everything on PVCs.
# Sizing math at our scale (~50 targets, 60s scrape): ~1-3 GB/day into Prometheus.
ssh "$K8S_CP_CONN" -- "cat > ~/monitoring-values.yaml <<'EOF'
prometheus:
  prometheusSpec:
    scrapeInterval: 1m
    evaluationInterval: 1m
    retention: 1d
    retentionSize: 8GB   # second guardrail: whichever hits first wins, disk never fills silently
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 10Gi } }
    resources: { requests: { cpu: 300m, memory: 1Gi } }
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 2Gi } }
grafana:
  persistence:
    enabled: true
    storageClassName: local-path
    size: 5Gi   # dashboards + sqlite survive restarts (emptyDir would wipe them)
    # NOTE: admin password is NOT in this file — passed via --set from Doppler at install time.
EOF"
```

### Step 2: Install the release (called `monitoring`, like the namespace)

```bash
# Helm CLI first (RKE2 ships the controller, not the CLI — installed once
# per CP; the `||` skips it when `helm version` already answers):
ssh "$K8S_CP_CONN" -- "helm version >/dev/null 2>&1 || curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash"
# Save the Grafana admin password in Doppler yourself first (any config, e.g.
# dev, key GRAFANA_ADMIN_PASSWORD), then pass it with the splice — and if it
# contains `$`, pre-escape it exactly like the Traefik dashboard hash
# (ESCAPED=${VAR//$/\$}), same remote-shell rule.
# NOTE: Grafana takes the password in PLAINTEXT (it bcrypt-hashes internally).
# Traefik basicAuth (dashboard, Prometheus below) takes the full htpasswd line.
export GRAFANA_ADMIN=$(doppler secrets get GRAFANA_ADMIN_PASSWORD --plain --project astrolumina --config prd)
ssh "$K8S_CP_CONN" -- "helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update"
ssh "$K8S_CP_CONN" -- 'helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f ~/monitoring-values.yaml --set grafana.adminPassword='"$GRAFANA_ADMIN"
unset GRAFANA_ADMIN
```

### Step 3: Expose both UIs through Traefik (persistent, no port-forward)

Port-forward dies with your laptop/SSH session. IngressRoutes are cluster
objects in etcd — they survive laptop reboots, VM reboots, everything. Same
pattern as staging: plain HTTP on the `web` entryPoint (lab has no public
DNS, so no TLS — Grafana has its own login anyway), plus the
`kubernetes.io/ingress.class: traefik` annotation the RKE2 DaemonSet requires
(same lesson as `staging/52-ingressroute.yaml`).

On the laptop, `/etc/hosts` (same node IP as staging):

```
192.168.122.11 prometheus.k8s.astrolumina.ro
192.168.122.11 grafana.k8s.astrolumina.ro
```

Prometheus has no login of its own, so it gets Traefik basicAuth with the
same password — uniform credentials everywhere. Unlike Grafana (plaintext),
Traefik wants the full htpasswd line (`admin:$2y$...`) in the secret's
`users` key. Pull it from Doppler (`PROMETHEUS_AUTH`, `prd` config) and
create the secret imperatively first (the Middleware below references it;
the namespace already exists from step 2):

```bash
export PROMETHEUS_AUTH=$(doppler secrets get PROMETHEUS_AUTH --plain --project astrolumina --config prd)
ESCAPED_AUTH=${PROMETHEUS_AUTH//$/\$}
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic prometheus-auth -n monitoring --from-literal=users='"$ESCAPED_AUTH"' --dry-run=client -o yaml | kubectl apply -f -'
unset PROMETHEUS_AUTH ESCAPED_AUTH
```

Write the routes locally — quoted `'EOF'` heredoc so the backticks in
`Host()` stay literal — then pipe the file through ssh (never inside shell
quotes, so nothing expands on either side):

```bash
cat > /tmp/monitoring-routes.yaml <<'EOF'
# Both routes are HTTPS-only on websecure. HTTP is deliberately NOT served.
# `tls: {}` serves Traefik's default self-signed cert (lab has no public
# DNS, so Let's Encrypt cannot issue) — expect a browser cert warning.
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  entryPoints: [websecure]
  tls: {}
  routes:
    - match: Host(`grafana.k8s.astrolumina.ro`)
      kind: Rule
      services:
        - name: monitoring-grafana
          port: 80
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: prometheus
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  entryPoints: [websecure]
  tls: {}
  routes:
    - match: Host(`prometheus.k8s.astrolumina.ro`)
      kind: Rule
      middlewares:
        - name: prometheus-auth
      services:
        - name: monitoring-kube-prometheus-prometheus
          port: 9090
---
# BasicAuth for Prometheus (same password as Grafana). The `secret` value is
# the name of the generic Secret created above; its `users` key holds the
# full htpasswd line. Deliberately NOT attached to the Grafana route —
# Grafana has its own login, basicAuth on top would double-prompt.
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: prometheus-auth
  namespace: monitoring
spec:
  basicAuth:
    secret: prometheus-auth
EOF
cat /tmp/monitoring-routes.yaml | ssh "$K8S_CP_CONN" -- "kubectl apply -f -"
```

Then open `http://grafana.k8s.astrolumina.ro` (login `admin` / your Doppler
password in PLAINTEXT form — preinstalled dashboards under "Kubernetes /
Compute Resources / Cluster") and `http://prometheus.k8s.astrolumina.ro`
(login `admin` / the SAME password, via Traefik basicAuth — Status → Targets
shows what gets scraped). No tunnel needed, ever again.

### Step 4: Verify

```bash
ssh "$K8S_CP_CONN" -- "kubectl get pods -n monitoring"
ssh "$K8S_CP_CONN" -- "kubectl get ingressroute -n monitoring"
```

Wait for `Running` on all pods (takes 1-2 minutes on first image pull).
Fallback only (proves the pods serve even if Traefik misbehaves):

```bash
ssh -L 3000:localhost:3000 "$K8S_CP_CONN" -- kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

### Step 5: Day-2 operations

`helm upgrade` after editing values, `helm rollback` if you break
something. NOTE: `helm uninstall monitoring -n monitoring` deletes the
workloads but NOT the PVCs — Prometheus/Grafana data stays on the node disks
until you delete the PVCs explicitly. That is a feature here (survives
accidental uninstalls and all VM reboots), just remember it when you actually
want a clean slate: `kubectl delete pvc --all -n monitoring`.

NOTE (changing the Grafana admin password later): the password from
`GRAFANA_ADMIN_PASSWORD` is only consumed on a FRESH database — Grafana
keeps the old hash in its sqlite DB on the `monitoring-grafana` PVC, so
patching the secret alone changes nothing at login. Full reset: patch the
secret, scale the deploy to 0, delete the PVC, re-create it manually (Helm
chart PVCs do NOT auto-recreate — same 5Gi `local-path` RWO claim),
scale back to 1. If the old pod hangs in `Terminating`, force-delete it
(`kubectl delete pod <name> -n monitoring --force --grace-period=0`).

NOTE (repo strategy): this repo stays Kustomize forever. Real Helm charts for
our apps will live in the separate AstroLumina-Helm repo and get consumed
declaratively (RKE2 `HelmChart`/`HelmChartConfig`, later ArgoCD/Flux) — that
is the GitOps path, while this repo remains the plain-manifest source of truth.
