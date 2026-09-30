# Development — step-by-step deploy guide

Deploys the whole stack into namespace `astrolumina-dev`: 1 replica per
service, NodePort Services, no ingress (mirrors the Compose dev setup where
ports are published and no Traefik exists).

NodePorts: frontend `30080`, astrology `30301`, payment `30302`,
booking `30303`.

## 0. Prerequisites (have these ready before you start)

- RKE2 VMs up: 1 control-plane + 2 workers, all `Ready`.
- Two-machine model: `doppler` CLI (logged in) runs on your LAPTOP — it only
  reads secrets into shell variables. `kubectl` runs on the CONTROL PLANE
  only, so every `kubectl` below is prefixed with
  `ssh "$K8S_CP_CONN" -- "..."`. This repo is mounted live at `/mnt/k8s`
  on the control plane — edits you make here apply straight from there, no
  `git pull` needed on the VM.
- In Doppler: project `astrolumina`, config `dev`, filled with EVERY
  variable the stack needs — not just secrets, but also the former ConfigMap
  values: `NODE_ENV`, all `*_SERVER_PORT` / `*_SERVER_DNS`,
  `ASTROLOGICAL_API_URL`, `PAYMENT_API_URL`, `BOOKING_API_URL`, `STRIPE_PK`,
  `R2_BASE_URL`, `ASTROLOGER_API_URL/HOST`, `CALCOM_BASE_URL`,
  `STRIPE_API_VER`, `CORS_ORIGINS`. Browser-facing URLs must contain the real
  node IP (no `NODE_IP` placeholder works at runtime — resolve the IP first,
  put the final URLs in Doppler). It must also contain `GITHUB_USER`,
  `GITHUB_TOKEN` (PAT with `read:packages`) and `GITHUB_EMAIL` — the GHCR
  credentials used in step 3b. It must also contain the four image tags
  `FRONTEND_DOCKER_IMAGE_TAG`, `ASTROLOGY_API_DOCKER_IMAGE_TAG`,
  `BOOKING_API_DOCKER_IMAGE_TAG`, `PAYMENT_API_DOCKER_IMAGE_TAG` (all
  `latest` for dev — the exact tags pinned in step 3c). Plus, before step 4,
  export `K8S_SERVICE_TOKEN` with the service token of the `dev` config —
  one `export` in your shell, step 4 consumes it directly, no dashboard
  copy-paste involved.
- The images referenced by the Deployments are private on GHCR. Every
  Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  `01-ghcr-secret.yaml` ships as a placeholder; step 3b replaces it with
  the real secret built from the Doppler `GITHUB_USER` / `GITHUB_TOKEN`.

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

## 2. Install the Doppler operator (once per cluster rebuild)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml"
ssh "$K8S_CP_CONN" -- "kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s"
ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"
```

Expected: deployment becomes `Available`, the CRD exists. This creates the
`doppler-operator-system` namespace, the `DopplerSecret` CRD, RBAC and the
controller. Skip only if you already installed it on this exact cluster.

## 3. Deploy development (placeholders first)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/development"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-dev"
```

Expected: 4 pods created. They will show `ImagePullBackOff` / `ErrImagePull`
until step 3b replaces the `ghcr-secret` placeholder — that is normal.
If a pod is `CrashLoopBackOff` (pulled fine, then crashed), check why before
continuing:

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -n astrolumina-dev deploy/frontend --tail=30"
```

## 3b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

`01-ghcr-secret.yaml` applied in step 3 is a placeholder with fake
credentials, so kubelet cannot pull the private GHCR images yet. The real
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

Expected: pods leave `ImagePullBackOff` and reach `Running` (they still boot
with the `02-secrets.yaml` placeholder env values until step 4 syncs Doppler).
Do NOT commit the real credentials — `01-ghcr-secret.yaml` stays a placeholder
in git.

## 3c. Pin the images to the Doppler tags

The manifests ship every image as `:latest`. The real tag per service lives
in Doppler (config `dev`): `FRONTEND_DOCKER_IMAGE_TAG`,
`ASTROLOGY_API_DOCKER_IMAGE_TAG`, `BOOKING_API_DOCKER_IMAGE_TAG`,
`PAYMENT_API_DOCKER_IMAGE_TAG`. (A Deployment's `image:` field is static —
Kubernetes cannot read it from a Secret — so the tags are applied
imperatively here, same pattern as steps 3b and 4. `imagePullPolicy` stays
`Always`, so `:latest` still resolves fresh even before this step runs.)
`export` lines on the laptop, `set image` on the CP (tags are plain
version strings, double quotes are safe):

```bash
export FRONTEND_TAG=$(doppler secrets get FRONTEND_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
export ASTROLOGY_TAG=$(doppler secrets get ASTROLOGY_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
export BOOKING_TAG=$(doppler secrets get BOOKING_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
export PAYMENT_TAG=$(doppler secrets get PAYMENT_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
ssh "$K8S_CP_CONN" -- "kubectl set image deploy/frontend frontend=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-frontend:$FRONTEND_TAG -n astrolumina-dev"
ssh "$K8S_CP_CONN" -- "kubectl set image deploy/astrology-api astrology-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-astrologyapi:$ASTROLOGY_TAG -n astrolumina-dev"
ssh "$K8S_CP_CONN" -- "kubectl set image deploy/booking-api booking-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-bookingapi:$BOOKING_TAG -n astrolumina-dev"
ssh "$K8S_CP_CONN" -- "kubectl set image deploy/payment-api payment-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-paymentapi:$PAYMENT_TAG -n astrolumina-dev"
unset FRONTEND_TAG ASTROLOGY_TAG BOOKING_TAG PAYMENT_TAG
ssh "$K8S_CP_CONN" -- "kubectl rollout status deploy/frontend -n astrolumina-dev"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-dev"
```

Without the Doppler CLI, copy the 4 values from the dashboard and substitute
them for the `$..._TAG` variables above.

Expected: the Deployments restart on the exact tags from Doppler
(`ssh "$K8S_CP_CONN" -- "kubectl describe deploy/frontend -n astrolumina-dev | grep Image:"`
shows the tag). If a pod reports `ErrImagePull` with `manifest unknown`, the tag
in Doppler does not exist on GHCR — fix the value in Doppler and re-run.

## 4. Create the token Secret and let the operator sync

The token is deliberately NOT in git. `K8S_SERVICE_TOKEN` (exported in the
prerequisites) holds the service token for the `dev` config — consume it
directly, no dashboard involved (idempotent, safe to re-run):

```bash
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic doppler-token-dev -n doppler-operator-system --from-literal=serviceToken='"$K8S_SERVICE_TOKEN"' --dry-run=client -o yaml | kubectl apply -f -'
```

No `K8S_SERVICE_TOKEN` in this shell? Generate one without touching the
dashboard (uses your existing Doppler CLI login):

```bash
export K8S_SERVICE_TOKEN=$(doppler configs tokens create k8s-dev --project astrolumina --config dev --plain)
```

then re-run the `kubectl create secret` above. (Generated tokens pile up in
Doppler — revoke the ones you no longer use:
`doppler configs tokens revoke <token-id> --project astrolumina --config dev`.)

The 4 `DopplerSecret` resources from `03-doppler-secrets.yaml` are already in
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

Expected: `sk_test_` or `sk_live_`, NOT the placeholder text. Repeat the
spirit of this check for one key per Secret if you want to be thorough.

## 5. Remove the placeholders

Now that Doppler owns the Secrets, stop applying the fake ones:

1. Edit `development/kustomization.yaml` and delete the
   `- 02-secrets.yaml` line.
2. Re-apply:

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

## 7. If something is wrong

- Pod stuck in `ImagePullBackOff` / `ErrImagePull` after step 3b: the
  `ghcr-secret` is missing, still the placeholder, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-dev"`
  and `ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-dev --sort-by=.lastTimestamp | tail -10"`.
  Recreate the secret with the command from step 3b, then
  `ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-dev"`. (The old Compose
  equivalent was `echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER
  --password-stdin` — in K8s the kubelet needs the secret, not a node-local
docker login.)
- Pod `CrashLoopBackOff` with a "missing environment variable" log: the
  Doppler `dev` config lacks that key. Add it in Doppler, the operator syncs
  within seconds and restarts the pods by itself.
- `DopplerSecret` shows an error on `describe`: usually a wrong/expired
  service token or a wrong `project`/`config` name. Fix and re-apply, or
  recreate the token Secret (delete + `kubectl create secret ...` again).
- `ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"`
  shows nothing: first check with `-A` (maybe they landed in the app
  namespace — that means your checkout predates the kustomization namespace
  fix; pull latest and re-apply), then confirm the operator install from
  step 2 (`ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"`).
- `kubectl get secrets` / `get pods` with no `-n` only shows the `default`
  namespace — always pass `-n astrolumina-dev` for this stack.
- NodePort unreachable from your machine: the VMs' firewall is the usual
  suspect. On a node, `ss -tlnp | grep 30080` must show a listener; if it
  does, open the port in the hypervisor/cloud firewall.
- HPA shows `<unknown>` metrics: expected, metrics-server is not bundled
  with RKE2. Pods still run fine; install metrics-server only if you want
  autoscaling during tests.
