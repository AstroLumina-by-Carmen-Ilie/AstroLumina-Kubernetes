# Staging — step-by-step deploy guide

Deploys into namespace `astrolumina-staging`: blue + green variants at
1 replica each, Traefik `IngressRoute` on entryPoint `web` (plain HTTP),
single host `staging.k8s.astrolumina.ro` with `/api/*` prefix routing (mirrors
the Compose staging `routes.yml`). Blue is live by default.

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
   points at one Doppler config (here: `stg`) plus a service token, and
   declares which keys to sync into which Kubernetes Secret. Once the token
   Secret exists, the operator creates the `env-*` Secrets with the real
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

- Fresh RKE2 VMs: 1 control-plane + 2 workers, all `Ready` (you rebuild the
  machines between environments).
- Two-machine model: `doppler` CLI (logged in) runs on your LAPTOP — it only
  reads secrets into shell variables. `kubectl` runs on the CONTROL PLANE
  only, so every `kubectl` below is prefixed with
  `ssh "$K8S_CP_CONN" -- "..."`. This repo is mounted live at `/mnt/k8s`
  on the control plane — edits you make here apply straight from there, no
  `git pull` needed on the VM.
- In Doppler: project `astrolumina`, config `stg`, holding the complete
  runtime set for the stack (every key the Deployments reference — the
  manifests plus the `01-doppler-secrets.yaml` sync lists are the exact
  schema, so no separate list is needed here). Two groups of keys take part in the setup
  itself, and you will export each of them below:
  - `GITHUB_USER`, `GITHUB_TOKEN` (PAT with `read:packages`) and
    `GITHUB_EMAIL` — the GHCR pull credentials used in step 3b
    (`dockerconfigjson` needs all three: username, password, email).
  - `DOPPLER_SERVICE_TOKEN` — the service token of the `stg` config,
    consumed in step 4. No dashboard copy-paste anywhere: every value
    below comes from `doppler secrets get`.
- The images referenced by the Deployments are private on GHCR. Every
  blue/green Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  No pull secret ships in git; step 3b creates the real one from the
  `GITHUB_EMAIL`.

## 1. Connect to the control plane (laptop → CP)

`K8S_CP_CONN` (e.g. `ubuntu@192.168.122.10`) already lives in Doppler
(config `stg`). Export it once per shell, then run `kubectl` on the CP
through it. Quoting rule for every `ssh` below: the remote command sits in
DOUBLE quotes, so `$VARS` from `doppler` expand on your laptop (intended —
the CP never sees Doppler), while `|` pipes inside the quotes still execute
on the CP:

```bash
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config stg)
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

Placement contract for this environment: blue Deployments carry
`nodeAffinity` pinning them to worker-01
(`rke2-worker-daniel-pirvu-01`), green to worker-02
(`rke2-worker-daniel-pirvu-02`) — each color lives on its own worker and
nothing lands on the CP. Every Deployment runs a single replica and each
HPA scales its app 1–3 — staging is sized for realism on lab hardware, not
for load. Even at max burst both colors fit their workers comfortably.

## 2. Install the Doppler operator (once per cluster rebuild)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml"
ssh "$K8S_CP_CONN" -- "kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s"
ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"
```

Expected: deployment `Available`, CRD exists.

## 3. Deploy staging

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/staging"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-staging"
```

Expected: 8 pods created (`*-blue` and `*-green`). They will show
`ImagePullBackOff` / `ErrImagePull` until step 3b creates the real
`ghcr-secret` — that is normal. They will also sit in
`CreateContainerConfigError` / `Pending` until step 4, because no `env-*`
Secrets exist yet — also normal: kubelet retries on its own and the pods
start by themselves once Doppler syncs. If any pod is `CrashLoopBackOff` (pulled fine,
then crashed), inspect before continuing:

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -n astrolumina-staging deploy/frontend-blue --tail=30"
```

## 3b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

No pull secret exists yet (nothing in git provides one), so kubelet cannot
pull the private GHCR images yet. The real
`GITHUB_USER` / `GITHUB_TOKEN` / `GITHUB_EMAIL` already live in Doppler
(config `stg`) — build the real secret imperatively from them, zero
copy-paste (same pattern as `doppler-token-stg`;
it cannot be synced by the Doppler operator because a pull secret must be
`type: kubernetes.io/dockerconfigjson`, not `Opaque`):

The three `export` lines run on your LAPTOP (Doppler lives here). The
`kubectl create` runs on the CP — note the quote splice
`'...'"$VAR"'...'`: secret values expand locally, so tokens with `$`,
`!` or quotes survive the trip intact, while the `|` pipe still executes
remotely:

```bash
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config stg)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config stg)
export GITHUB_EMAIL=$(doppler secrets get GITHUB_EMAIL --plain --project astrolumina --config stg)
ssh "$K8S_CP_CONN" -- 'kubectl create secret docker-registry ghcr-secret -n astrolumina-staging --docker-server=ghcr.io --docker-username='"$GITHUB_USER"' --docker-password='"$GITHUB_TOKEN"' --docker-email='"$GITHUB_EMAIL"' --dry-run=client -o yaml | kubectl apply -f -'
unset GITHUB_TOKEN GITHUB_USER GITHUB_EMAIL
```

Then force a re-pull and verify:

```bash
ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-staging"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-staging"
ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-staging --sort-by=.lastTimestamp | tail -10"
```

Expected: pods leave `ImagePullBackOff` and reach `Running` (config still
missing until step 4 syncs Doppler — the `CreateContainerConfigError` note
in step 3 applies).
Do NOT commit the real credentials — create them only with the command
above, never in a file.

## 3c. Image tags are pinned in git (blue AND green)

Image tags are the source of truth in git — the same model as the Compose
`versions.env` files (the staging `versions.env` holds the same values, so
keep them in parity). Doppler no longer carries any `*_DOCKER_IMAGE_TAG`
keys. Each Deployment's `image:` field already references its pinned tag
(frontend `2.0.8`, astrology-api `2.0.8`, booking-api `2.0.7`, payment-api
`2.0.8`), so applying the manifests deploys those exact tags on both colors
— there is nothing else to run.

To promote a new build, bump the tag in the blue AND green Deployment files
(keep both colors identical unless you are staging a release on the idle
color), commit, and re-apply:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/staging && kubectl get pods -n astrolumina-staging"
```

Expected: all 8 Deployments run the tags from git
(`ssh "$K8S_CP_CONN" -- "kubectl describe deploy/frontend-blue -n astrolumina-staging | grep Image:"`
shows `:2.0.8`). If a pod reports `ErrImagePull` with `manifest unknown`,
the tag does not exist on GHCR — fix the value in the manifests and re-apply.

## 4. Create the token Secret and let the operator sync

The token is deliberately NOT in git. It already lives in Doppler as
`DOPPLER_SERVICE_TOKEN` (per-environment value, `stg` config) — pull it into
your shell, then consume it (idempotent, safe to re-run):

```bash
export DOPPLER_SERVICE_TOKEN=$(doppler secrets get DOPPLER_SERVICE_TOKEN --plain --project astrolumina --config stg)
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic doppler-token-stg -n doppler-operator-system --from-literal=serviceToken='"$DOPPLER_SERVICE_TOKEN"' --dry-run=client -o yaml | kubectl apply -f -'
```

If `doppler secrets get DOPPLER_SERVICE_TOKEN` ever reports the key missing
(e.g. revoked upstream and never re-saved), mint a fresh one without
touching the dashboard (uses your existing Doppler CLI login) and store it
back in Doppler under the same name, so this step stays zero-paste:

```bash
export DOPPLER_SERVICE_TOKEN=$(doppler configs tokens create k8s-stg --project astrolumina --config stg --plain)
```

then re-run the `kubectl create secret` above. (Generated tokens pile up in
Doppler — revoke the ones you no longer use:
`doppler configs tokens revoke <token-id> --project astrolumina --config stg`.)

Verify the sync (prints only the first 8 chars of one key):

```bash
ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"
ssh "$K8S_CP_CONN" -- "kubectl describe dopplersecret astrolumina-staging-payment-api -n doppler-operator-system"
ssh "$K8S_CP_CONN" -- "kubectl get secret env-payment-api-secrets -n astrolumina-staging -o jsonpath='{.data.STRIPE_SK}' | base64 -d | cut -c1-8"
```

Expected: no errors on `describe`; the key starts with `sk_test_` (staging
uses test keys).

## 5. Prove the build is sync-safe (nothing to remove)

In the old layout this step deleted placeholder lines from
`kustomization.yaml`. That logic is gone: `01`/`02` are no longer listed,
so re-applying is a pure no-op on secrets — Doppler keeps owning the
values and ArgoCD can sync freely. Prove it:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/staging"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-staging"
```

Expected: all 8 pods restart (via the `secrets.doppler.com/reload`
annotation) and return to `Running`.

## 6. Reach it from your host

Traefik gets a node IP from the bundled Klipper ServiceLB. Get one (on the CP):

```bash
ssh "$K8S_CP_CONN" -- "kubectl get nodes -o wide"
```

Add to `/etc/hosts` on your local machine (use a Traefik worker IP,
e.g. 192.168.122.11):

```text
192.168.122.11 staging.k8s.astrolumina.ro dashboard.k8s.astrolumina.ro
```

These `*.k8s.astrolumina.ro` names coexist with the Compose `127.0.0.1`
entries — no context toggling needed for them.

Then open `http://staging.k8s.astrolumina.ro` in the browser, and check the API
routes directly (these `curl` runs stay on your LAPTOP — they test your
host → node path):

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://staging.k8s.astrolumina.ro/api/astrology/<health-path>
curl -s -o /dev/null -w '%{http_code}\n' http://staging.k8s.astrolumina.ro/api/booking/<health-path>
curl -s -o /dev/null -w '%{http_code}\n' http://staging.k8s.astrolumina.ro/api/payment/<health-path>
```

(Replace `<health-path>` with each API's real health endpoint.) The
`/api/*` prefix is stripped before reaching the pods, exactly like Compose.

## 7. Blue-green cutover practice (staging is where you rehearse this)

Blue serves traffic now. To switch to green:

1. Confirm green is healthy: `ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-staging -l variant=green"`.
2. Smoke-test green directly (bypass Traefik). `port-forward` must bind on
your laptop, so the `ssh` carries a `-L` tunnel and the remote
`port-forward` binds to the CP's localhost:
```bash
ssh -L 8080:localhost:8080 "$K8S_CP_CONN" -- kubectl port-forward -n astrolumina-staging deploy/frontend-green 8080:80
```
then open `http://localhost:8080` on your laptop.
3. Flip the 4 `selector: variant: blue` fields to `green` in
`53-live-services.yaml` (edit here — the CP sees it live at `/mnt/k8s`)
and `ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/staging"`.
4. Re-run the step 6 curl checks. Roll back by flipping back to `blue`.

Keep the idle color running until the new one proves healthy.

## 8. If something is wrong

- Pod stuck in `ImagePullBackOff` / `ErrImagePull` after step 3b: the
  `ghcr-secret` is missing, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-staging"`
  and `ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-staging --sort-by=.lastTimestamp | tail -10"`.
  Recreate the secret with the command from step 3b, then
  `ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-staging"`.
- `curl` returns 404 on `/api/*`: the request never matched the IngressRoute.
  Check `ssh "$K8S_CP_CONN" -- "kubectl get ingressroute -n astrolumina-staging"`
  and that your `/etc/hosts` points at a node IP (not the VM hostname).
- Pod `CrashLoopBackOff` with missing env: add the key to the Doppler `stg`
  config; sync + restart are automatic.
- `describe dopplersecret` shows auth errors: wrong token or wrong
  `project`/`config` in `01-doppler-secrets.yaml`.
- `ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"`
  shows nothing: the operator install from step 2 did not complete
  (confirm with `ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"`),
  or the token Secret is missing or misnamed — the DopplerSecrets must land
  in `doppler-operator-system` (re-run the check with `-A` to see which
  namespace yours went to).
- `kubectl get secrets` / `get pods` with no `-n` only shows the `default`
  namespace — always pass `-n astrolumina-staging` for this stack.
- HPA `<unknown>` metrics: metrics-server is not bundled with RKE2; harmless
  for testing.
