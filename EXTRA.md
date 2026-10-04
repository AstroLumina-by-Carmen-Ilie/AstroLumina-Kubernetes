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

### Helm use-cases, generally

1. **Installing third-party software** — Prometheus, Grafana, cert-manager,
   ingress controllers. Someone else maintains the templates; you supply only
   values.
2. **Packaging your own apps for reuse** — one chart deployed as many
   releases (per customer, per region, per developer), each with different
   values.
3. **Versioned releases with rollback** — every `helm upgrade` stores a
   revision server-side; a bad deploy is one `helm rollback`, not archaeology
   in git history.
4. **Parameterized complexity** — conditionals and loops over values
   (optional TLS block, N replicas from a list, per-tenant resources) that
   plain YAML cannot express.
5. **Distribution** — charts are shareable artifacts (registries,
   `helm pull`), with provenance and signing.

### How this repo could become a Helm chart (cluster untouched)

Same topology, Helm packaging:

```
charts/astrolumina/
  Chart.yaml                 # name, version, appVersion
  values-dev.yaml            # NodePorts, tags, permissive policy
  values-staging.yaml        # Traefik hosts, HPA 1-3, strict policy
  values-production.yaml     # + TLS, dashboard auth, HPA 3-5
  templates/
    namespace.yaml
    frontend-deployment.yaml # replicas/image from {{ .Values }}
    service.yaml
    doppler-secret.yaml      # DopplerSecret CRDs, key lists from values
    networkpolicy.yaml       # {{- if .Values.traefik.enabled }} for the Traefik peer
    ingressroute.yaml        # {{- if .Values.traefik.enabled }}, hosts/TLS from values
    hpa.yaml
    helpers.tpl              # shared labels, written by hand — no transformer surprises
    NOTES.txt                # post-install hints (hosts entries, dashboard URL)
```

- **Blue/green** becomes two releases (`helm install app-blue … --set
  color=blue`) or one release ranging over both colors. The `variant` label
  stays — it just takes its value from `.Values`.
- **What would NOT move into Helm**: the Doppler *values* (still the secret
  source), the GHCR pull secret and token Secret (still imperative one-liners
  from Doppler), and the RKE2 `HelmChartConfig` driving Traefik itself (that
  layer stays underneath).
- **Sane migration path**: shape the chart → `helm template -f
  values-staging.yaml` → `diff` against today's `kubectl kustomize` output
  until byte-identical → only then switch the apply path. During transition,
  Kustomize can stay on as `--post-renderer` for last-mile tweaks.
- **When it is NOT worth it**: small team, three environments, no reuse —
  explicit Kustomize files beat chart abstraction. Helm pays off at reuse
  scale (many releases) or when per-release rollback matters.

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

### Why this repo is shaped like this

- **One kustomization per environment** (`development/`, `staging/`,
  `production/`), because environments differ in *composition*, not just
  values: dev has NodePort Services and no Traefik files at all;
  staging/prod add middlewares, IngressRoutes and live-services.
- **The env kustomization holds everything shared**: namespace, secrets wiring
  (`01` placeholder, `02` placeholder, `03` Doppler sync), Traefik pieces,
  network policy, HPA. It answers "what does this environment consist of" —
  the numbered prefixes (`00-`, `01-`, …, `70-`) keep boot order readable.
- **Blue/green are sub-kustomizations** (`staging/blue/`, `staging/green/`)
  holding only the 4 Deployments each. The color rides on a hardcoded
  `variant: blue|green` label (selectors, pod templates), and the traffic
  switch lives in `53-live-services.yaml`, whose selectors point at one color.
  Promoting = flipping those selectors, not re-rendering anything.
- **No `namespace:` and no `commonLabels` at any level** — both transformers
  proved harmful here (namespace relocated the DopplerSecrets; labels
  poisoned NetworkPolicy peers and froze Deployment selectors).

### Could it be shaped differently? Yes — trade-offs

- **One kustomization with all 16 Deployments inline**: fewer files, but the
  env file becomes a wall of entries and the blue/green boundary (which
  deploys belong to a color?) turns implicit. The split keeps per-color
  `apply`/delete trivial.
- **Classic base + overlays** (one `base/` plus `overlays/dev|stg|prd` patching
  replicas, images, namespaces): the textbook style, great when environments
  differ only by small tweaks. Here the differences are structural (Traefik
  present/absent, secrets wiring, live-services), so overlays would be mostly
  full-file replacements — more indirection for the same file count.
- **`namePrefix: blue-`/`green-` instead of variant labels**: the
  Kustomize-native way to tell copies apart, but the live-services switch and
  the `-l variant=blue` log queries need a *label*, and prefixes rename
  everything (harder to eyeball). Labels won.
- **`images:` transformer for tags instead of imperative `set image`**: cleaner
  and reproducible — but tags live in Doppler (source of truth outside git),
  so a patch would duplicate them into git and drift. The imperative step won.

### Kustomize use-cases, generally

1. **First-party apps × N environments** — this repo. One source, per-env
   composition.
2. **Last-mile patching of third-party YAML**: `helm template … | kustomize
   build` — render a chart, then patch what values cannot express
   (annotations, extra labels, resource tweaks) without forking the chart.
3. **GitOps rendering**: the applied artifact is plain YAML, so PR diffs show
   exactly what the cluster will get. No server-side state to reason about.
4. **Environment bootstrapping**: namespaces, RBAC, operators, CRDs composed
   per cluster profile.
   It stops fitting when you need loops/conditionals ("one Deployment per
   tenant from a list") or versioned rollback of a whole release — that is
   Helm territory.

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

## 4. Monitoring: Prometheus + Grafana via Helm

Moved to [MONITORING.md](./MONITORING.md) — install steps, storage backend,
Traefik routes (`prometheus.k8s.astrolumina.ro`, `grafana.k8s.astrolumina.ro`),
verify, and day-2 operations.
