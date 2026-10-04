# EXTRA: Helm, Kustomize, Monitoring

Companion to the root `README.md`. This repo (manifests + Kustomize) is the
"day-1 platform"; this file explains the two packaging tools around it and
the observability stack. Written from what we verified live on our RKE2
cluster — not from docs alone.

## 1. What is Helm and how to use it

**Helm = the package manager for Kubernetes** ("apt/yum for clusters").

- A **chart** is a package: Go templates + `values.yaml` (defaults) +
  `Chart.yaml` (metadata). Example: `prometheus-community/kube-prometheus-stack`.
- A **release** is an installed instance of a chart. The same chart can yield
  N releases (`monitoring`, `monitoring-test`, …), each with its own values.
- You configure a release without touching templates:
  `-f my-values.yaml` or `--set key=value`.
- Helm keeps state (as Secrets, in the release's namespace), so it knows
  **upgrade + rollback**: `helm upgrade …`, `helm rollback <release> <rev>`,
  `helm history <release>`.

Typical flow with a third-party chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm show values prometheus-community/kube-prometheus-stack | head -50  # what you can configure
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f values.yaml
helm status monitoring -n monitoring
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
helm rollback monitoring -n monitoring 2
helm uninstall monitoring -n monitoring
```

### Helm on RKE2 — what "bundled" really means

RKE2 ships the Helm **controller**, not the CLI. Verified live:
`/var/lib/rancher/rke2/bin/` holds `kubectl`, `crictl`, `ctr` — no `helm`.
What exists instead are the `helmcharts` / `helmchartconfigs.helm.cattle.io`
CRDs: you drop a `HelmChart`/`HelmChartConfig` manifest into
`/var/lib/rancher/rke2/server/manifests/` and the controller installs the
chart server-side. That is exactly how our Traefik runs — and the
`traefik-config.yaml` you already edited (Let's Encrypt resolver,
`persistence.enabled`) is a HelmChartConfig, i.e. you have been driving Helm
all along, just declaratively.

So "Helm bundled with RKE2" = engine without a steering wheel. For
interactive commands (`repo`, `status`, `rollback`, ad-hoc installs) install
the CLI once:

```bash
ssh "$K8S_CP_CONN" -- "curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash && helm version"
```

Recommendation: keep the CLI as a tool; for permanent cluster components
(Traefik and the like) prefer HelmChart manifests in git, as before.

### Where your AstroLumina-Helm repo fits

Charts you author yourself (one per app, or an umbrella chart for the whole
stack) live there. Two ways to consume them on RKE2:

1. **Imperative (CLI):** `helm install astrolumina ./charts/astrolumina -f values-prod.yaml`
   — fast iteration, full rollback support, state in the release Secrets.
2. **Declarative (RKE2-native):** a `HelmChart` manifest pointing at your
   chart + a `HelmChartConfig` with the values, dropped in the manifests dir
   — GitOps-style, the controller reconciles it like it does Traefik.

Both read the same `values-*.yaml`. Start with (1) while iterating on the
charts, graduate to (2) when they stabilize.

## 2. What is Kustomize and where it is useful

**Kustomize = customize plain YAML without templates.** No `{{ }}` anywhere:
you start from base manifests and layer transformations on top:

- **resources:** compose files per environment (`kustomization.yaml`).
- **patches:** targeted edits (strategic-merge or JSON patches) for what
  differs between dev/staging/prod.
- **transformers:** cross-cutting rewrites (`namespace:`, `namePrefix:`,
  `images:` for tags, `commonLabels:`).
- **generators:** build ConfigMaps/Secrets from literals or files.

It keeps **no state** in the cluster and has no server: you render locally
(`kustomize build`, built into kubectl) and apply the output. That is what
this repo uses — one `kustomization.yaml` per environment composing the same
Deployments/Services/Secrets/policies/Traefik routes with per-env tweaks.

Where it shines: **your own manifests across environments** (exactly this
repo). Where it does not: packaging third-party software — you would be
copy-pasting someone else's YAML and maintaining the fork forever. That is
Helm's job.

Hard-won rule from this cluster: `commonLabels` are injected INTO selectors
too — including NetworkPolicy peers. A peer asking for labels the target
does not carry matches nothing and traffic dies silently. That is why this
repo carries no `commonLabels` at all; identity labels live hardcoded in the
manifests instead.

## 3. The difference, interview version

Both solve "how do YAML manifests get into the cluster", from opposite
directions:

- Kustomize says **"take MY YAML and tweak it"**. Best for first-party
  manifests, N environments, GitOps-friendly (rendered output is reviewable
  diff). No releases, no rollback — `kubectl` + git history is the audit trail.
- Helm says **"take SOMEBODY'S PACKAGE and configure it"**. Best for
  third-party software (Prometheus, Grafana, cert-manager). Releases are
  versioned server-side, rollback is one command.

They complement each other, and even combine: `helm template … | kustomize`
(Helm renders, Kustomize patches last-mile details), or Helm's `--post-renderer`.
Rule of thumb: Helm for what you **install**, Kustomize for what you **own**.

### Cheat-sheet

```bash
# Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update && helm repo list
helm search repo prometheus-stack
helm show values prometheus-community/kube-prometheus-stack | head -50  # what you can configure
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f values.yaml
helm list -A && helm status monitoring -n monitoring
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
helm history monitoring -n monitoring && helm rollback monitoring -n monitoring 2
helm get values monitoring -n monitoring   # what values are live
helm template myrel chart/ -f values.yaml   # render without installing (template dry-run)
helm uninstall monitoring -n monitoring

# Kustomize (built into kubectl, nothing to install)
kubectl kustomize /mnt/k8s/development      # render, do not apply (check)
kubectl apply -k /mnt/k8s/development       # render + apply
kubectl diff -k /mnt/k8s/staging            # what would change, without applying
```

## 4. Monitoring: Prometheus + Grafana via Helm (namespace `monitoring`)

The `kube-prometheus-stack` chart brings Prometheus + Grafana + Alertmanager +
exporters (node-exporter, kube-state-metrics) in a single release.
Two lab quirks, both verified on our cluster:

1. **Zero StorageClasses** in the cluster → default persistence would wedge
   pods in Pending (exactly the Traefik PVC incident). That is why the values
   below force `emptyDir` everywhere: metric data is lost on pod restart —
   acceptable in the lab, NOT in prod (there you need real storage).
2. **Small nodes (2 vCPU)** → the stack is hungry (~1 Gi mem for Prometheus
   alone). Watch with `kubectl top pods -n monitoring`; if it gets tight,
   the first knob is `retention` (how much history Prometheus keeps).

Steps (run on the laptop, with the Doppler exports done):

```bash
# 1. values on the CP (quoted 'EOF' heredoc: nothing expands locally)
ssh "$K8S_CP_CONN" -- "cat > ~/monitoring-values.yaml <<'EOF'
prometheus:
  prometheusSpec:
    retention: 48h
    storageSpec: {}          # emptyDir: no StorageClass in the lab, the PVC would stay Pending
                                        # (prod: replace with a volumeClaimTemplate on real storage)
alertmanager:
  alertmanagerSpec:
    storage: {}             # same reason as above
grafana:
  persistence:
    enabled: false          # no PVC; custom dashboards are lost on restart (lab-only!)
    adminPassword: 'change-me-on-first-login'   # LAB-ONLY in plaintext; in prod the password comes from secret management
EOF"

# 2. repo + install (the release is called `monitoring`, like the namespace)
ssh "$K8S_CP_CONN" -- "helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update"
ssh "$K8S_CP_CONN" -- "helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f ~/monitoring-values.yaml"

# 3. verify
ssh "$K8S_CP_CONN" -- "kubectl get pods -n monitoring"
```

Wait for `Running` on all of them (takes 1-2 minutes on first image pull).
Then Grafana, via port-forward (the service is called `<release>-grafana`):

```bash
ssh -L 3000:localhost:3000 "$K8S_CP_CONN" -- kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

open `http://localhost:3000`, login `admin` / the password from values, and
you get preinstalled dashboards ("Kubernetes / Compute Resources / Cluster").
Prometheus UI the same way: `svc/monitoring-kube-prometheus-stack-prometheus 9090:9090`
(targets: Status → Targets — that is where you see what gets scraped).

Minimal day-2: `helm upgrade` after editing values, `helm rollback` if you
break something, `helm uninstall monitoring -n monitoring` deletes everything
(the `monitoring` namespace stays — delete it separately if you want).
