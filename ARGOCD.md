# ArgoCD — GitOps for Production

> Production-grade guide for this lab (RKE2, Traefik, Kustomize). Read the
> prerequisites first — pointing ArgoCD at production with the repo in its
> current shape would actively break it. Theory, then install, then wiring.

---

## 0. Prerequisites (do these BEFORE creating the production app)

ArgoCD syncs **exactly what is in git**. Three things about this repo's
current shape would hurt production on the very first sync:

1. **Placeholders still wired in `production/`.** The prod kustomization
   still lists `01-ghcr-secret.yaml` and `02-secrets.yaml`. Today you
   avoid them by applying files selectively; ArgoCD does not do
   selective — it applies the whole kustomization. First sync would
   overwrite the live pull secret and the live app secrets with
   placeholders. Complete production step 5 first (remove **both** lines),
   verify with `grep -n "01-\|02-" production/kustomization.yaml`
   returning nothing.
2. **Manifests say `:latest`, production runs pinned tags.** The prod
   Deployment files declare `image: ...:latest`, while live runs the
   pinned builds (frontend `2.0.7`, astrology `2.0.7`, booking `2.0.6`,
   payment `2.0.7`). The first sync would "fix" that drift — e.g. booking
   would jump `2.0.6` → `:latest` (which is `2.0.7`, the build that
   dropped routes). Commit the pinned tags into the production manifests
   so git and the cluster agree; from then on an image upgrade is a
   commit changing a tag, and rollback is `git revert`. Stop running
   `kubectl set image` by hand once ArgoCD owns the namespace — it would
   revert your edit within minutes.
3. **Everything must be committed and pushed.** ArgoCD reads the git
   remote, not your laptop's mount. `git status` must be clean (or at
   least: everything ArgoCD should see is pushed). Local-only edits are
   invisible to it.

Doppler side stays as it is: ArgoCD syncs the `DopplerSecret` CRs and the
operator resolves the values. The `prd` config must be complete, same as
for the manual flow.

---

## 1. What is ArgoCD?

ArgoCD is a **declarative GitOps delivery tool** for Kubernetes. The core
idea is one sentence:

> **Git is the single source of truth; ArgoCD makes the cluster match it.**

The traditional flow (what you do today) is imperative: you run
`kubectl apply -f ...` from your laptop. The cluster ends up in some state,
but nothing guarantees it still matches the repo tomorrow — anyone with
`kubectl` (or a `--dry-run` typo, or a forgotten `-f`) can drift it.
Production lived exactly this risk when manifests were re-applied with
`:latest` over the pinned builds.

The GitOps flow flips this:

1. You commit manifests to git (Kustomize, Helm, or plain YAML).
2. ArgoCD **watches** the repo and the cluster, continuously.
3. When they differ, it **syncs** — applies the repo state to the cluster.
4. The ArgoCD UI shows every app green (in sync) or red (drift / failure).

Concretely, ArgoCD gives you:

- **Drift detection** — manual `kubectl` edits show up as `OutOfSync`
  instead of silently rotting.
- **Self-healing** — with automated sync + `selfHeal: true`, ArgoCD
  reverts manual changes automatically. The cluster becomes effectively
  read-only for humans. (Note: `selfHeal` only acts under automated
  sync; with manual sync it does nothing — see §4b.)
- **Pruning** — with `prune: true`, deleting a file from git deletes the
  object from the cluster. No more orphaned Services like the ones we
  cleaned up by hand.
- **History + rollback** — every sync is tied to a git SHA. Rollback is
  `git revert`, not archaeology.
- **Multi-environment from one repo** — one `Application` per directory:
  `development/`, `staging/`, `production/` map 1:1 to ArgoCD apps.

### ArgoCD vs `kubectl apply` (interview version)

| `kubectl apply -k` from laptop | ArgoCD |
|---|---|
| Imperative, runs when you remember | Declarative, reconciles continuously |
| Drift invisible | Drift visible + optionally auto-fixed |
| No audit trail (who applied what) | Every change = git commit |
| Secrets/placeholders handled by convention | Same, but enforced by sync discipline |
| Fine for a lab learning curve | What production GitOps looks like |

ArgoCD does **not** replace Kustomize or Helm — it *consumes* them. It runs
`kustomize build` or `helm template` server-side and applies the result.
Your repo stays exactly as it is today; ArgoCD just becomes the thing that
applies it instead of your laptop.

---

## 2. How to install it

Helm chart (`argo/argo-cd`), namespace `argocd`, applied from your laptop
through the usual ssh pattern. No storage needed (ArgoCD keeps its state
in its own Redis + etcd-backed CRDs; in a lab, chart defaults are fine).

```bash
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config prd)

# Helm CLI first (RKE2 ships the controller, not the CLI — installed once
# per CP; the `||` skips it when `helm version` already answers):
ssh "$K8S_CP_CONN" -- "helm version >/dev/null 2>&1 || curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash"

# Chart repo + pick a version to pin (never install unpinned in prod):
ssh "$K8S_CP_CONN" -- "helm repo add argo https://argoproj.github.io/argo-helm && helm repo update"
ssh "$K8S_CP_CONN" -- "helm search repo argo/argo-cd --versions | head -5"
```

Take the newest version from that list and put it in `CHART_VERSION` below.
Values (`~/argocd-values.yaml` on the CP — one file, two decisions):

```bash
cat <<'EOF' | ssh "$K8S_CP_CONN" -- "cat > ~/argocd-values.yaml"
# Traefik IngressRoute (see below) replaces the chart's own Ingress.
server:
  ingress:
    enabled: false
  # ArgoCD server listens HTTPS on 8080 by default. Traefik terminates TLS
  # at the edge and talks plaintext to the backend (same as every other
  # route here), so the server must serve plain HTTP — otherwise the UI
  # route 502s. In-cluster only; the browser still gets HTTPS.
  extraArgs:
    - --insecure
EOF
```

```bash
ssh "$K8S_CP_CONN" -- "helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace --version <CHART_VERSION> -f ~/argocd-values.yaml"

# Wait for it.
ssh "$K8S_CP_CONN" -- "kubectl rollout status deploy/argocd-server -n argocd --timeout=300s"
```

> The chart bundles its CRDs (`Application`, `AppProject`, ...). Minor
> upgrades are hands-free; across major chart versions, check the upstream
> upgrade notes — CRDs sometimes need a manual refresh.

### First login

The initial admin password is generated into a Secret (plaintext, base64):

```bash
ssh "$K8S_CP_CONN" -- "kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d; echo"
```

Username is `admin`. Change it after first login
(`argocd account update-password`, or via the UI under User Info).

### Expose the UI through Traefik (HTTPS-only, like everything else)

Production and monitoring are HTTPS-only in this cluster, so the ArgoCD UI
gets a `websecure` route with TLS (default Traefik cert in the lab —
same as Grafana/Prometheus). No Traefik basicAuth in front: ArgoCD has
its own login, so like Grafana (and unlike Prometheus) the app login is
the gate.

```bash
cat <<'EOF' | ssh "$K8S_CP_CONN" -- "kubectl apply -f -"
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`argocd.k8s.astrolumina.ro`)
      kind: Rule
      services:
        - name: argocd-server
          port: 80
  tls: {}
EOF
```

Laptop `/etc/hosts`:

```
192.168.122.11 argocd.k8s.astrolumina.ro
```

Then open `https://argocd.k8s.astrolumina.ro/` (accept the lab default
cert, same as the other HTTPS routes).

> The heredoc is quoted (`<<'EOF'`) so the backticks in `Host()` survive
> the laptop shell; the pipe runs `kubectl apply` on the CP. Same trick as
> the monitoring routes.

---

## 3. What is it good for? (and not for)

**Good for:**

- Owning the deploy step end-to-end: commit → ArgoCD syncs → done.
- Blue/green and canary discipline: ArgoCD shows both colors, their health,
  and which one the live Service points at.
- Multi-cluster: one ArgoCD can deliver the same repo to many clusters
  (dev cluster, prod cluster) with per-cluster overrides.
- Day-2 visibility: which commit is live where, what drifted, what failed
  to sync and why — without ssh-ing anywhere.

**Not for:**

- Building images or running CI tests — that is CI (GitHub Actions,
  Jenkins). ArgoCD is CD-only: it deploys what CI already built.
- Managing secrets content — it syncs `Secret` objects like anything else,
  but the *values* still come from your flow (Doppler operator, Sealed
  Secrets, External Secrets). Keep the current Doppler setup; ArgoCD just
  applies the `DopplerSecret` CRs and lets the operator fill them in.
- Emergency surgery — when the cluster is on fire you still `kubectl`
  directly, then reconcile git afterwards.

---

## 4. Pointing it at a repo

### 4a. Register the repository

**Public repo (HTTPS, no credentials):** register it directly — either in
the UI (Settings → Repositories → Connect Repo) or declaratively:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f -" <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: astrolumina-kubernetes
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/<org>/AstroLumina-Kubernetes.git
EOF
```

**Private repo (SSH):** generate a deploy key, add the public half as a
GitHub deploy key (read-only), store the private half in the same Secret
shape with `sshPrivateKey` instead of nothing. ArgoCD uses it for
`git clone` only — it never pushes.

### 4b. Create the production Application (staging first, production last)

An `Application` is a small CR: *where to read* (repo + path + revision)
× *where to write* (cluster + namespace) × *how to sync*.

Production rules differ from the other envs on purpose:

- `targetRevision` pins a **tag**, never `HEAD`. What is live in prod
  must be a named, reviewable pointer — `HEAD` means "whatever landed
  last", which is a staging habit, not a production one.
- Sync starts **manual**. You press SYNC in the UI per change until the
  flow feels boring. Only then switch to automated. (`selfHeal` only
  acts under automated sync — with manual sync it is inert, so don't
  expect drift correction before you flip the switch.)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: astrolumina-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/AstroLumina-Kubernetes.git
    targetRevision: main   # a tag, NOT head
    path: production                  # directory inside the repo
  destination:
    server: https://kubernetes.default.svc   # the local cluster
    namespace: astrolumina-prod
  syncPolicy:
    prune: true                 # delete objects removed from git
    # no `automated` block yet = manual sync from the UI.
    # Graduate later to: automated: { prune: true, selfHeal: true }
```

Apply it once from the laptop; ArgoCD takes over from there:

```bash
cat <app-above>.yaml | ssh "$K8S_CP_CONN" -- "kubectl apply -f -"
```

Create the **staging** Application first (same shape, `path: staging`,
`targetRevision` can track the staging branch), watch one full change
(commit → sync → green), then production. Development last — it changes
too often to deserve ceremony early.

For blue/green promotion the flow stays git-native: flip the
`variant: blue|green` selector in `53-live-services.yaml`, tag, update
`targetRevision`, sync.

### 4c. What ArgoCD expects to find (supported formats)

ArgoCD inspects the `path` and picks a toolchain automatically, in this
order:

1. **Kustomize** — a `kustomization.yaml` in the directory.
   This is what this repo already is per environment, so each env works
   as an ArgoCD app with zero restructuring.
2. **Helm** — a `Chart.yaml` in the directory (it runs
   `helm template` with `values.yaml` + any `values-*.yaml`).
   This is the shape a future `AstroLumina-Helm` repo would take.
3. **Plain directory** — bare `*.yaml` files applied in bulk
   (no templating, no ordering beyond filename sorting).
4. **Jsonnet / plugins** — niche; ignore unless you need them.

Practical rules for the layout:

- One tool per directory. Do not mix `Chart.yaml` and
  `kustomization.yaml` in the same path — ArgoCD picks Kustomize and
  silently ignores the chart.
- Keep the per-environment directories self-contained
  (`development/`, `staging/`, `production/`), exactly like now.
- Secrets content stays out of git, unchanged: ArgoCD syncs the
  `DopplerSecret` CRs and the operator resolves the values.
- Image tags for production live **in git** (see §0). An upgrade is a
  commit, a rollback is a revert — never `kubectl set image` once
  ArgoCD owns the namespace.

---

## 5. Day-2 operations (cheat-sheet)

```bash
# Sync status of everything, one shot.
ssh "$K8S_CP_CONN" -- "kubectl get applications -n argocd"

# Full detail on one app (health, sync state, live commit SHA).
ssh "$K8S_CP_CONN" -- "kubectl describe application astrolumina-production -n argocd"

# Force a refresh + sync from the CLI (needs the argocd CLI or use the UI button).
# UI: open the app → REFRESH → SYNC.

# Roll back = move targetRevision to the previous tag (or git revert the
# offending commit), ArgoCD re-syncs by itself.

# Temporarily stop ArgoCD from touching an app (manual surgery window):
# UI → app Details → SYNC POLICY → disable Auto-Sync. Re-enable after.
```

### Suggested path for this project

1. Stabilize staging + production on the manual flow first (you are here).
2. Finish §0 (placeholders out, images pinned in git, everything pushed).
3. Install ArgoCD, register the repo, create the **staging** Application
   only. Watch one full change (commit → sync → green) before trusting it.
4. Create the **production** Application with a pinned tag and manual
   sync. Promote staging → production by moving the tag. Enable
   automated sync + selfHeal only when the promotion flow feels boring.
5. Later: move chart-able pieces into the `AstroLumina-Helm` repo and
   point ArgoCD apps at Helm paths instead of Kustomize ones —
   per-directory, one at a time, same mechanism.
