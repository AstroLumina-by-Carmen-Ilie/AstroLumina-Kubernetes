# ArgoCD — GitOps for Kubernetes

> Practical guide for this lab (RKE2, Traefik, Kustomize). Theory first,
> then install, then wiring it to a repo.

---

## 1. What is ArgoCD?

ArgoCD is a **declarative GitOps delivery tool** for Kubernetes. The core
idea is one sentence:

> **Git is the single source of truth; ArgoCD makes the cluster match it.**

The traditional flow (what you do today) is imperative: you run
`kubectl apply -f ...` from your laptop. The cluster ends up in some state,
but nothing guarantees it still matches the repo tomorrow — anyone with
`kubectl` (or a `--dry-run` typo, or a forgotten `-f`) can drift it.

The GitOps flow flips this:

1. You commit manifests to git (Kustomize, Helm, or plain YAML).
2. ArgoCD **watches** the repo and the cluster, continuously.
3. When they differ, it **syncs** — applies the repo state to the cluster.
4. The ArgoCD UI shows every app green (in sync) or red (drift / failure).

Concretely, ArgoCD gives you:

- **Drift detection** — manual `kubectl` edits show up as `OutOfSync`
  instead of silently rotting.
- **Self-healing** — with `selfHeal: true`, ArgoCD reverts manual changes
  automatically. The cluster becomes effectively read-only for humans.
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

Upstream manifests, namespace `argocd`, applied from your laptop through the
usual ssh pattern. No storage needed (ArgoCD keeps its state in its own
Redis + etcd-backed CRDs; in a lab, defaults are fine).

```bash
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config dev)

# 1. Namespace + official manifests (stable tag — pin it, do not use latest).
ssh "$K8S_CP_CONN" -- "kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -"
ssh "$K8S_CP_CONN" -- "kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.3/manifests/install.yaml"

# 2. Wait for it.
ssh "$K8S_CP_CONN" -- "kubectl rollout status deploy/argocd-server -n argocd --timeout=300s"
```

### First login

The initial admin password is generated into a Secret (plaintext, base64):

```bash
ssh "$K8S_CP_CONN" -- "kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d; echo"
```

Username is `admin`. Change it after first login
(`argocd account update-password`, or via the UI under User Info).

### Expose the UI through Traefik (persistent, no port-forward)

Same pattern as the monitoring routes: an IngressRoute in the app's own
namespace, on the `web` entryPoint, with the ingress-class annotation.
ArgoCD serves plain HTTP internally by default (`argocd-server:80`);
TLS terminates at Traefik in production.

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
    - web
  routes:
    - match: Host(`argocd.k8s.astrolumina.ro`)
      kind: Rule
      services:
        - name: argocd-server
          port: 80
EOF
```

Laptop `/etc/hosts`:

```
192.168.122.11 argocd.k8s.astrolumina.ro
```

Then open `http://argocd.k8s.astrolumina.ro/`.

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

### 4b. Create one Application per environment

An `Application` is a small CR: *where to read* (repo + path + revision)
× *where to write* (cluster + namespace) × *how to sync*.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: astrolumina-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/AstroLumina-Kubernetes.git
    targetRevision: HEAD          # or a branch/tag/SHA to pin
    path: staging                # directory inside the repo
  destination:
    server: https://kubernetes.default.svc   # the local cluster
    namespace: astrolumina-staging
  syncPolicy:
    automated:
      prune: true                 # delete objects removed from git
      selfHeal: true              # revert manual kubectl edits
```

Apply it once from the laptop; ArgoCD takes over from there:

```bash
cat <app-above>.yaml | ssh "$K8S_CP_CONN" -- "kubectl apply -f -"
```

Repeat with `path: development` / `path: production` for the other two.
For blue/green promotion the flow stays git-native: flip the
`variant: blue|green` selector in `53-live-services.yaml`, commit, ArgoCD
syncs the switch.

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
- Files ArgoCD must **not** apply (placeholders like `01-*` / `02-*`)
  should either be removed from git (the step-5 flow) or excluded via
  `spec.syncPolicy.syncOptions: Replace=false` / `ignoreDifferences`
  — otherwise ArgoCD will faithfully re-apply the placeholders over
  your live secrets, the same way a careless `apply -k` would.
- Secrets content stays out of git, unchanged: ArgoCD syncs the
  `DopplerSecret` CRs and the operator resolves the values.

---

## 5. Day-2 operations (cheat-sheet)

```bash
# Sync status of everything, one shot.
ssh "$K8S_CP_CONN" -- "kubectl get applications -n argocd"

# Full detail on one app (health, sync state, live commit SHA).
ssh "$K8S_CP_CONN" -- "kubectl describe application astrolumina-staging -n argocd"

# Force a refresh + sync from the CLI (needs the argocd CLI or use the UI button).
# UI: open the app → REFRESH → SYNC.

# Roll back = git revert the offending commit, ArgoCD re-syncs by itself.

# Temporarily stop ArgoCD from touching an app (manual surgery window):
# UI → app Details → SYNC POLICY → disable Auto-Sync. Re-enable after.
```

### Suggested path for this project

1. Stabilize staging + production on the manual flow first (you are here).
2. Install ArgoCD, register the repo, create the **staging** Application
   only. Watch one full change (commit → auto-sync → green) before
   trusting it.
3. Add development, then production last — production with
   `selfHeal: true` but sync triggered manually from the UI at first
   (remove `automated` until the promotion flow feels boring).
4. Later: move chart-able pieces into the `AstroLumina-Helm` repo and
   point ArgoCD apps at Helm paths instead of Kustomize ones —
   per-directory, one at a time, same mechanism.
