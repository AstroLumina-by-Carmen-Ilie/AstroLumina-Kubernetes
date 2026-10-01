# Staging — step-by-step deploy guide

Deploys into namespace `astrolumina-staging`: blue + green variants at
3 replicas each, Traefik `IngressRoute` on entryPoint `web` (plain HTTP),
single host `staging.astrolumina.ro` with `/api/*` prefix routing (mirrors
the Compose staging `routes.yml`). Blue is live by default.

## 0. Prerequisites (have these ready before you start)

- Fresh RKE2 VMs: 1 control-plane + 2 workers, all `Ready` (you rebuild the
  machines between environments).
- Two-machine model: `doppler` CLI (logged in) runs on your LAPTOP — it only
  reads secrets into shell variables. `kubectl` runs on the CONTROL PLANE
  only, so every `kubectl` below is prefixed with
  `ssh "$K8S_CP_CONN" -- "..."`. This repo is mounted live at `/mnt/k8s`
  on the control plane — edits you make here apply straight from there, no
  `git pull` needed on the VM.
- In Doppler: project `astrolumina`, config `stg`, filled with EVERY
  variable the stack needs (shared vars, frontend/API URLs with the
  `staging.astrolumina.ro` host, `CORS_ORIGINS`, plus all secrets). It must
  also contain `GITHUB_USER`, `GITHUB_TOKEN` (PAT with `read:packages`) and
  `GITHUB_EMAIL` — the GHCR credentials used in step 3b. It must also contain
  the four image tags `FRONTEND_DOCKER_IMAGE_TAG` (2.0.6),
  `ASTROLOGY_API_DOCKER_IMAGE_TAG` (2.0.6), `BOOKING_API_DOCKER_IMAGE_TAG`
  (2.0.5), `PAYMENT_API_DOCKER_IMAGE_TAG` (2.0.6) — the exact tags pinned
  in step 3c. It also holds `K8S_SERVICE_TOKEN` — the service token of the
  `stg` config. Step 4 pulls it straight from Doppler, no dashboard
  copy-paste anywhere.
- The images referenced by the Deployments are private on GHCR. Every
  blue/green Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  `01-ghcr-secret.yaml` ships as a placeholder; step 3b replaces it with
  the real secret built from the Doppler `GITHUB_USER` / `GITHUB_TOKEN`.

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

## 2. Install the Doppler operator (once per cluster rebuild)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml"
ssh "$K8S_CP_CONN" -- "kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s"
ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"
```

Expected: deployment `Available`, CRD exists.

## 3. Deploy staging (placeholders first)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/staging"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-staging"
```

Expected: 8 pods created (`*-blue` and `*-green`). They will show
`ImagePullBackOff` / `ErrImagePull` until step 3b replaces the `ghcr-secret`
placeholder — that is normal. If any pod is `CrashLoopBackOff` (pulled fine,
then crashed), inspect before continuing:

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -n astrolumina-staging deploy/frontend-blue --tail=30"
```

## 3b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

`01-ghcr-secret.yaml` applied in step 3 is a placeholder with fake
credentials, so kubelet cannot pull the private GHCR images yet. The real
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

Expected: pods leave `ImagePullBackOff` and reach `Running` (they still boot
with the `02-secrets.yaml` placeholder env values until step 4 syncs Doppler).
Do NOT commit the real credentials — `01-ghcr-secret.yaml` stays a placeholder
in git.

## 3c. Pin the images to the Doppler tags (blue AND green)

The manifests ship every image as `:latest`. The real tag per service lives
in Doppler (config `stg`): `FRONTEND_DOCKER_IMAGE_TAG` (2.0.6),
`ASTROLOGY_API_DOCKER_IMAGE_TAG` (2.0.6), `BOOKING_API_DOCKER_IMAGE_TAG`
(2.0.5), `PAYMENT_API_DOCKER_IMAGE_TAG` (2.0.6). (A Deployment's `image:`
field is static — Kubernetes cannot read it from a Secret — so the tags are
applied imperatively here, same pattern as steps 3b and 4.) `export` lines
on the laptop, `set image` on the CP (tags are plain version strings,
double quotes are safe). Pin both colors:

```bash
export FRONTEND_TAG=$(doppler secrets get FRONTEND_DOCKER_IMAGE_TAG --plain --project astrolumina --config stg)
export ASTROLOGY_TAG=$(doppler secrets get ASTROLOGY_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config stg)
export BOOKING_TAG=$(doppler secrets get BOOKING_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config stg)
export PAYMENT_TAG=$(doppler secrets get PAYMENT_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config stg)
for COLOR in blue green; do
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/frontend-$COLOR frontend=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-frontend:$FRONTEND_TAG -n astrolumina-staging"
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/astrology-api-$COLOR astrology-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-astrologyapi:$ASTROLOGY_TAG -n astrolumina-staging"
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/booking-api-$COLOR booking-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-bookingapi:$BOOKING_TAG -n astrolumina-staging"
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/payment-api-$COLOR payment-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-paymentapi:$PAYMENT_TAG -n astrolumina-staging"
done
unset FRONTEND_TAG ASTROLOGY_TAG BOOKING_TAG PAYMENT_TAG
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-staging"
```

Without the Doppler CLI, copy the 4 values from the dashboard and substitute
them for the `$..._TAG` variables above.

Expected: all 8 Deployments restart on the exact tags from Doppler
(`ssh "$K8S_CP_CONN" -- "kubectl describe deploy/frontend-blue -n astrolumina-staging | grep Image:"`
shows `:2.0.6`). If a pod reports `ErrImagePull` with `manifest unknown`,
the tag in Doppler does not exist on GHCR — fix the value in Doppler and
re-run. Run this again on BOTH colors every time you promote a new build.

## 4. Create the token Secret and let the operator sync

The token is deliberately NOT in git. It already lives in Doppler as
`K8S_SERVICE_TOKEN` (per-environment value, `stg` config) — pull it into
your shell, then consume it (idempotent, safe to re-run):

```bash
export K8S_SERVICE_TOKEN=$(doppler secrets get K8S_SERVICE_TOKEN --plain --project astrolumina --config stg)
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic doppler-token-stg -n doppler-operator-system --from-literal=serviceToken='"$K8S_SERVICE_TOKEN"' --dry-run=client -o yaml | kubectl apply -f -'
```

If `doppler secrets get K8S_SERVICE_TOKEN` ever reports the key missing
(e.g. revoked upstream and never re-saved), mint a fresh one without
touching the dashboard (uses your existing Doppler CLI login) and store it
back in Doppler under the same name, so this step stays zero-paste:

```bash
export K8S_SERVICE_TOKEN=$(doppler configs tokens create k8s-stg --project astrolumina --config stg --plain)
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
uses test keys), not the placeholder text.

## 5. Remove the placeholders (both files, not just 02)

1. Edit `staging/kustomization.yaml` and delete BOTH lines:
   `- 02-secrets.yaml` AND `- 01-ghcr-secret.yaml`. Leaving either one
   listed means the next `apply -k` overwrites a live secret with its
   placeholder — `02` clobbers the Doppler-synced values, `01` clobbers the
   real pull secret and the following rollout dies with `ImagePullBackOff`.
   (Git keeps the originals — a fresh rebuild starts from placeholders again.)
2. If this step was already run the old way, re-run step 3b FIRST to restore
   the real `ghcr-secret`, then re-apply and confirm the automatic restart:

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

Add to `/etc/hosts` on your local machine (replace with the real node IP):

```text
192.168.1.10 staging.astrolumina.ro
```

Then open `http://staging.astrolumina.ro` in the browser, and check the API
routes directly (these `curl` runs stay on your LAPTOP — they test your
host → node path):

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://staging.astrolumina.ro/api/astrology/<health-path>
curl -s -o /dev/null -w '%{http_code}\n' http://staging.astrolumina.ro/api/booking/<health-path>
curl -s -o /dev/null -w '%{http_code}\n' http://staging.astrolumina.ro/api/payment/<health-path>
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
ssh -L 8080:localhost:8080 "$K8S_CP_CONN" -- kubectl port-forward -n astrolumina-staging svc/frontend-green 8080:80
```
then open `http://localhost:8080` on your laptop.
3. Flip the 4 `selector: variant: blue` fields to `green` in
`53-live-services.yaml` (edit here — the CP sees it live at `/mnt/k8s`)
and `ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/staging"`.
4. Re-run the step 6 curl checks. Roll back by flipping back to `blue`.

Keep the idle color running until the new one proves healthy.

## 8. If something is wrong

- Pod stuck in `ImagePullBackOff` / `ErrImagePull` after step 3b: the
  `ghcr-secret` is missing, still the placeholder, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-staging"`
  and `ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-staging --sort-by=.lastTimestamp | tail -10"`.
  Recreate the secret with the command from step 3b, then
  `ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-staging"`. (The old Compose
  equivalent was `echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER
  --password-stdin` — in K8s the kubelet needs the secret, not a node-local
docker login.)
- `curl` returns 404 on `/api/*`: the request never matched the IngressRoute.
  Check `ssh "$K8S_CP_CONN" -- "kubectl get ingressroute -n astrolumina-staging"`
  and that your `/etc/hosts` points at a node IP (not the VM hostname).
- Pod `CrashLoopBackOff` with missing env: add the key to the Doppler `stg`
  config; sync + restart are automatic.
- `describe dopplersecret` shows auth errors: wrong token or wrong
  `project`/`config` in `03-doppler-secrets.yaml`.
- `ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"`
  shows nothing: first check with `-A` (maybe they landed in the app
  namespace — that means your checkout predates the kustomization namespace
  fix; pull latest and re-apply), then confirm the operator install from
  step 2 (`ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"`).
- `kubectl get secrets` / `get pods` with no `-n` only shows the `default`
  namespace — always pass `-n astrolumina-staging` for this stack.
- HPA `<unknown>` metrics: metrics-server is not bundled with RKE2; harmless
  for testing.
