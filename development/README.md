# Development — step-by-step deploy guide

Deploys the whole stack into namespace `astrolumina-dev`: 1 replica per
service, NodePort Services, no ingress (mirrors the Compose dev setup where
ports are published and no Traefik exists).

NodePorts: frontend `30080`, astrology `30301`, payment `30302`,
booking `30303`.

## How this environment boots (read first)

Everything below assumes an **empty cluster** — no operator, no secrets,
nothing yet. The deploy works in two phases:

1. **Kustomize creates everything except secrets.** The first
   `kubectl apply -k` creates the namespace, Deployments, Services and
   `DopplerSecret` objects — but NO `ghcr-secret` and NO `env-*` Secrets
   (nothing in git provides them). The pods are created and wait:
   `ImagePullBackOff` until step 3b creates the real pull secret, and
   `CreateContainerConfigError` / `Pending` until step 4 gives the operator
   its token. Both states are expected, not errors — kubelet retries on
   its own and the pods recover by themselves once the secrets exist.
2. **Doppler fills in the real values.** The Doppler Kubernetes operator
   watches the `DopplerSecret` objects (`01-doppler-secrets.yaml`). Each one
   points at one Doppler config (here: `dev`) plus a service token, and
   declares which keys to sync into which Kubernetes Secret. Once the token
   Secret exists, the operator creates the `env-*` Secrets with the real
   values and rolls the pods
   (`secrets.doppler.com/reload` annotation) — no redeploy needed.
   Two secrets cannot come from Doppler and are created imperatively
   instead: `ghcr-secret` (a pull secret must be
   `type: kubernetes.io/dockerconfigjson`, while the operator only syncs
   `Opaque`) and the token Secret itself (the operator needs the token to
   authenticate — chicken-and-egg).
There is NO third phase anymore: no placeholder files are listed in
   `kustomization.yaml` (`01`/`02` exist in the directory only as inert
   samples), so every future `apply -k` — and every ArgoCD sync — touches
   manifests and `DopplerSecret` objects only, never the live secret
   values.

## 0. Prerequisites (have these ready before you start)

- RKE2 VMs up: 1 control-plane + 2 workers, all `Ready`.
- Two-machine model: `doppler` CLI (logged in) runs on your LAPTOP — it only
  reads secrets into shell variables. `kubectl` runs on the CONTROL PLANE
  only, so every `kubectl` below is prefixed with
  `ssh "$K8S_CP_CONN" -- "..."`. This repo is mounted live at `/mnt/k8s`
  on the control plane — edits you make here apply straight from there, no
  `git pull` needed on the VM.
- In Doppler: project `astrolumina`, config `dev`, holding the complete
  runtime set for the stack (every key the Deployments reference — the
  manifests plus the `01-doppler-secrets.yaml` sync lists are the exact
  schema, so no separate list is needed here). Two groups of keys take part in the setup
  itself, and you will export each of them below:
  - `GITHUB_USER`, `GITHUB_TOKEN` (PAT with `read:packages`) and
    `GITHUB_EMAIL` — the GHCR pull credentials used in step 3b
    (`dockerconfigjson` needs all three: username, password, email).
  - `DOPPLER_SERVICE_TOKEN` — the service token of the `dev` config,
    consumed in step 4. No dashboard copy-paste anywhere: every value
    below comes from `doppler secrets get`.
- The images referenced by the Deployments are private on GHCR. Every
  Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  No pull secret ships in git; step 3b creates the real one from the
  `GITHUB_EMAIL`.

## 1. Connect to the control plane (laptop → CP)

`K8S_CP_CONN` (e.g. `ubuntu@192.168.122.10`) already lives in Doppler
(config `dev`). Export it once per shell, then run `kubectl` on the CP
through it. Quoting rule for every `ssh` below: the remote command sits in
DOUBLE quotes, so `$VARS` from `doppler` expand on your laptop (intended —
the CP never sees Doppler), while `|` pipes inside the quotes still execute
on the CP:

```bash
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config dev)
ssh "$K8S_CP_CONN" -- "kubectl get nodes"
```

If `kubectl: command not found` (fresh VM — RKE2 hides its binary outside
the default PATH and ships no kubeconfig for your user), run this once per
control plane — afterwards plain `kubectl` works in every `ssh` below:

```bash
ssh "$K8S_CP_CONN" -- "sudo ln -sf /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl && mkdir -p ~/.kube && sudo cat /etc/rancher/rke2/rke2.yaml > ~/.kube/config && chmod 600 ~/.kube/config && kubectl get nodes"
```

Expected: 3 nodes, all `Ready`. If not, stop here and fix the VMs first.

Then taint the control plane so app pods never schedule there (fresh RKE2
leaves the CP untainted and schedulable). One command, idempotent, safe to
re-run — system pods (CoreDNS, canal, Traefik) already tolerate this
standard taint, app pods don't:

```bash
ssh "$K8S_CP_CONN" -- "kubectl taint nodes -l node-role.kubernetes.io/control-plane=true node-role.kubernetes.io/control-plane=:NoSchedule --overwrite"
```

Placement contract for this environment: dev pods carry no node affinity —
the taint alone keeps them on the two workers, wherever the scheduler fits
them.

## 2. Install the Doppler operator (once per cluster rebuild)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml"
ssh "$K8S_CP_CONN" -- "kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s"
ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"
```

Expected: deployment becomes `Available`, the CRD exists. This creates the
`doppler-operator-system` namespace, the `DopplerSecret` CRD, RBAC and the
controller. Skip only if you already installed it on this exact cluster.

## 3. Deploy development

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/development"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-dev"
```

Expected: 4 pods created. They will show `ImagePullBackOff` / `ErrImagePull`
until step 3b creates the real `ghcr-secret` — that is normal. They will
also sit in `CreateContainerConfigError` / `Pending` until step 4, because
no `env-*` Secrets exist yet — also normal: kubelet retries on its own
and the pods start by themselves once Doppler syncs.
If a pod is `CrashLoopBackOff` (pulled fine, then crashed), check why before
continuing:

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -n astrolumina-dev deploy/frontend --tail=30"
```

## 3b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

No pull secret exists yet (nothing in git provides one), so kubelet cannot
pull the private GHCR images yet. The real
`GITHUB_USER` / `GITHUB_TOKEN` / `GITHUB_EMAIL` already live in Doppler
(config `dev`) — build the real secret imperatively from them, zero
copy-paste (same pattern as `doppler-token-dev`; it cannot be synced by the
Doppler operator because a pull secret must be
`type: kubernetes.io/dockerconfigjson`, not `Opaque`):

The three `export` lines run on your LAPTOP (Doppler lives here). The
`kubectl create` runs on the CP — note the quote splice
`'...'"$VAR"'...'`: secret values expand locally, so tokens with `$`,
`!` or quotes survive the trip intact, while the `|` pipe still executes
remotely:

```bash
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config dev)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config dev)
export GITHUB_EMAIL=$(doppler secrets get GITHUB_EMAIL --plain --project astrolumina --config dev)
ssh "$K8S_CP_CONN" -- 'kubectl create secret docker-registry ghcr-secret -n astrolumina-dev --docker-server=ghcr.io --docker-username='"$GITHUB_USER"' --docker-password='"$GITHUB_TOKEN"' --docker-email='"$GITHUB_EMAIL"' --dry-run=client -o yaml | kubectl apply -f -'
unset GITHUB_TOKEN GITHUB_USER GITHUB_EMAIL
```

Then force a re-pull and verify:

```bash
ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-dev"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-dev"
ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-dev --sort-by=.lastTimestamp | tail -10"
```

Expected: pods leave `ImagePullBackOff` and reach `Running` (config still
missing until step 4 syncs Doppler — the `CreateContainerConfigError` note
in step 3 applies).
Do NOT commit the real credentials — create them only with the command
above, never in a file.

## 3c. Image tags are pinned in git

Image tags are the source of truth in git — the same model as the Compose
`versions.env` files (the development `versions.env` holds the same values,
currently `latest` everywhere). Doppler no longer carries any
`*_DOCKER_IMAGE_TAG` keys. Each Deployment's `image:` field already
references its tag, so applying the manifests is all it takes
(`imagePullPolicy` stays `Always`, so `:latest` still resolves fresh on
every pull).

To try a pinned build, bump the tag in the 4 Deployment files, commit, and
re-apply:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/development && kubectl get pods -n astrolumina-dev"
```

Expected: the Deployments run the tags from git. If a pod reports
`ErrImagePull` with `manifest unknown`, the tag does not exist on GHCR —
fix the value in the manifests and re-apply.

## 4. Create the token Secret and let the operator sync

The token is deliberately NOT in git. It already lives in Doppler as
`DOPPLER_SERVICE_TOKEN` (per-environment value, `dev` config) — pull it into
your shell, then consume it (idempotent, safe to re-run):

```bash
export DOPPLER_SERVICE_TOKEN=$(doppler secrets get DOPPLER_SERVICE_TOKEN --plain --project astrolumina --config dev)
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic doppler-token-dev -n doppler-operator-system --from-literal=serviceToken='"$DOPPLER_SERVICE_TOKEN"' --dry-run=client -o yaml | kubectl apply -f -'
```

If `doppler secrets get DOPPLER_SERVICE_TOKEN` ever reports the key missing
(e.g. revoked upstream and never re-saved), mint a fresh one without
touching the dashboard (uses your existing Doppler CLI login) and store it
back in Doppler under the same name, so this step stays zero-paste:

```bash
export DOPPLER_SERVICE_TOKEN=$(doppler configs tokens create k8s-dev --project astrolumina --config dev --plain)
```

then re-run the `kubectl create secret` above. (Generated tokens pile up in
Doppler — revoke the ones you no longer use:
`doppler configs tokens revoke <token-id> --project astrolumina --config dev`.)

The 4 `DopplerSecret` resources from `01-doppler-secrets.yaml` are already in
the cluster (applied in step 3). The operator now syncs the `dev` config.
Verify:

```bash
ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"
ssh "$K8S_CP_CONN" -- "kubectl describe dopplersecret astrolumina-dev-payment-api -n doppler-operator-system"
```

Expected: no errors in `describe` output. Then confirm a real value landed
(prints only the first 8 chars, never paste full secrets in chat/logs):

```bash
ssh "$K8S_CP_CONN" -- "kubectl get secret env-payment-api-secrets -n astrolumina-dev -o jsonpath='{.data.STRIPE_SK}' | base64 -d | cut -c1-8"
```

Expected: `sk_test_` or `sk_live_`. Repeat the
spirit of this check for one key per Secret if you want to be thorough.

## 5. Prove the build is sync-safe (nothing to remove)

In the old layout this step deleted placeholder lines from
`kustomization.yaml`. That logic is gone: `01`/`02` are no longer listed,
so re-applying is a pure no-op on secrets — Doppler keeps owning the
values and ArgoCD can sync freely. Prove it:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/development"
ssh "$K8S_CP_CONN" -- "kubectl rollout status deploy/frontend -n astrolumina-dev"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-dev"
```

Expected: pods restart automatically (the `secrets.doppler.com/reload`
annotation tells the operator to roll them) and come back `Running`.

## 6. Access from your host browser

Find a node IP (`ssh "$K8S_CP_CONN" -- "kubectl get nodes -o wide"`), then open:

- Frontend: `http://<node-ip>:30080`
- Astrology API: `http://<node-ip>:30301`
- Payment API: `http://<node-ip>:30302`
- Booking API: `http://<node-ip>:30303`

The frontend will load, but its API calls will fail out of the box:
`env.js` carries `http://astrology-api:3031`-style URLs — cluster-internal
DNS names your laptop browser cannot resolve. They are fixed in Doppler.
The contract is: the manifest reads the app-facing names
(`ASTROLOGY_API_URL` / `PAYMENT_API_URL` / `BOOKING_API_URL`) from the
`*_K8S_URL` secret keys (`development/10-frontend-deployment.yaml`), so the
`dev` config must hold `ASTROLOGY_API_K8S_URL`, `PAYMENT_API_K8S_URL`,
`BOOKING_API_K8S_URL` with the K8s-reachable endpoints
(`http://<node-ip>:30301/30302/30303` — any node IP works, NodePort
listens on all nodes).

Watch the frontend pod roll (the `secrets.doppler.com/reload` annotation
restarts it once the operator re-syncs), then hard-refresh the browser page
(`env.js` is cached aggressively).

## 7. If something is wrong

- Pod stuck in `ImagePullBackOff` / `ErrImagePull` after step 3b: the
  `ghcr-secret` is missing, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-dev"`
  and `ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-dev --sort-by=.lastTimestamp | tail -10"`.
  Recreate the secret with the command from step 3b, then
  `ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-dev"`.
- Pod `CrashLoopBackOff` with a "missing environment variable" log: the
  Doppler `dev` config lacks that key. Add it in Doppler, the operator syncs
  within seconds and restarts the pods by itself.
- `DopplerSecret` shows an error on `describe`: usually a wrong/expired
  service token or a wrong `project`/`config` name. Fix and re-apply, or
  recreate the token Secret (delete + `kubectl create secret ...` again).
- `ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"`
  shows nothing: the operator install from step 2 did not complete
  (confirm with `ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"`),
  or the token Secret is missing or misnamed — the DopplerSecrets must land
  in `doppler-operator-system` (re-run the check with `-A` to see which
  namespace yours went to).
- `kubectl get secrets` / `get pods` with no `-n` only shows the `default`
  namespace — always pass `-n astrolumina-dev` for this stack.
- NodePort unreachable from your machine: the VMs' firewall is the usual
  suspect. On a node, `ss -tlnp | grep 30080` must show a listener; if it
  does, open the port in the hypervisor/cloud firewall.
- HPA shows `<unknown>` metrics: expected, metrics-server is not bundled
  with RKE2. Pods still run fine; install metrics-server only if you want
  autoscaling during tests.
