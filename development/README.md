# Development — step-by-step deploy guide

Deploys the whole stack into namespace `astrolumina-dev`: 1 replica per
service, NodePort Services, no ingress (mirrors the Compose dev setup where
ports are published and no Traefik exists).

NodePorts: frontend `30080`, astrology `30301`, payment `30302`,
booking `30303`.

## 0. Prerequisites (have these ready before you start)

- RKE2 VMs up: 1 control-plane + 2 workers, all `Ready`.
- `kubectl` on your local machine plus the kubeconfig of the cluster.
- In Doppler: project `astrolumina`, config `dev`, filled with EVERY
  variable the stack needs — not just secrets, but also the former ConfigMap
  values: `NODE_ENV`, all `*_SERVER_PORT` / `*_SERVER_DNS`,
  `ASTROLOGICAL_API_URL`, `PAYMENT_API_URL`, `BOOKING_API_URL`, `STRIPE_PK`,
  `R2_BASE_URL`, `ASTROLOGER_API_URL/HOST`, `CALCOM_BASE_URL`,
  `STRIPE_API_VER`, `CORS_ORIGINS`. Browser-facing URLs must contain the real
  node IP (no `NODE_IP` placeholder works at runtime — resolve the IP first,
  put the final URLs in Doppler). It must also contain `GITHUB_USER` and
  `GITHUB_TOKEN` (PAT with `read:packages`) — the GHCR credentials used in
  step 3b. It must also contain the four image tags
  `FRONTEND_DOCKER_IMAGE_TAG`, `ASTROLOGY_API_DOCKER_IMAGE_TAG`,
  `BOOKING_API_DOCKER_IMAGE_TAG`, `PAYMENT_API_DOCKER_IMAGE_TAG` (all
  `latest` for dev — the exact tags pinned in step 3c). Plus the **service token** for the `dev` config (Doppler dashboard
  -> project -> `dev` -> Access).
- The images referenced by the Deployments are private on GHCR. Every
  Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  `01-ghcr-secret.yaml` ships as a placeholder; step 3b replaces it with
  the real secret built from the Doppler `GITHUB_USER` / `GITHUB_TOKEN`.

## 1. Point kubectl at the fresh cluster

```bash
export KUBECONFIG=/path/to/rke2.yaml
kubectl get nodes
```

Expected: 3 nodes, all `Ready`. If not, stop here and fix the VMs first.

## 2. Install the Doppler operator (once per cluster rebuild)

```bash
kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml
kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s
kubectl get crd dopplersecrets.secrets.doppler.com
```

Expected: deployment becomes `Available`, the CRD exists. This creates the
`doppler-operator-system` namespace, the `DopplerSecret` CRD, RBAC and the
controller. Skip only if you already installed it on this exact cluster.

## 3. Deploy development (placeholders first)

```bash
cd AstroLumina-Kubernetes
kubectl apply -k development/
kubectl get pods -n astrolumina-dev
```

Expected: 4 pods created. They will show `ImagePullBackOff` / `ErrImagePull`
until step 3b replaces the `ghcr-secret` placeholder — that is normal.
If a pod is `CrashLoopBackOff` (pulled fine, then crashed), check why before
continuing:

```bash
kubectl logs -n astrolumina-dev deploy/frontend --tail=30
```

## 3b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

`01-ghcr-secret.yaml` applied in step 3 is a placeholder with fake
credentials, so kubelet cannot pull the private GHCR images yet. The real
`GITHUB_USER` / `GITHUB_TOKEN` already live in Doppler (config `dev`) —
build the real secret imperatively (same pattern as `doppler-token-dev`;
it cannot be synced by the Doppler operator because a pull secret must be
`type: kubernetes.io/dockerconfigjson`, not `Opaque`):

```bash
kubectl create secret docker-registry ghcr-secret \
  -n astrolumina-dev \
  --docker-server=ghcr.io \
  --docker-username='<paste-GITHUB_USER-from-Doppler-dev>' \
  --docker-password='<paste-GITHUB_TOKEN-from-Doppler-dev>' \
  --docker-email='admin@astrolumina.com' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Or straight from the Doppler CLI (no copy-paste, token never stays in shell
history if you unset it right after):

```bash
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config dev)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config dev)
kubectl create secret docker-registry ghcr-secret -n astrolumina-dev \
  --docker-server=ghcr.io --docker-username="$GITHUB_USER" \
  --docker-password="$GITHUB_TOKEN" --docker-email='admin@astrolumina.com' \
  --dry-run=client -o yaml | kubectl apply -f -
unset GITHUB_TOKEN GITHUB_USER
```

Then force a re-pull and verify:

```bash
kubectl rollout restart deploy -n astrolumina-dev
kubectl get pods -n astrolumina-dev
kubectl get events -n astrolumina-dev --sort-by=.lastTimestamp | tail -10
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
For dev those keys hold `latest`; read them and set the images:

```bash
export FRONTEND_TAG=$(doppler secrets get FRONTEND_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
export ASTROLOGY_TAG=$(doppler secrets get ASTROLOGY_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
export BOOKING_TAG=$(doppler secrets get BOOKING_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
export PAYMENT_TAG=$(doppler secrets get PAYMENT_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config dev)
kubectl set image deploy/frontend frontend=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-frontend:$FRONTEND_TAG -n astrolumina-dev
kubectl set image deploy/astrology-api astrology-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-astrologyapi:$ASTROLOGY_TAG -n astrolumina-dev
kubectl set image deploy/booking-api booking-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-bookingapi:$BOOKING_TAG -n astrolumina-dev
kubectl set image deploy/payment-api payment-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-paymentapi:$PAYMENT_TAG -n astrolumina-dev
unset FRONTEND_TAG ASTROLOGY_TAG BOOKING_TAG PAYMENT_TAG
kubectl rollout status deploy/frontend -n astrolumina-dev
kubectl get pods -n astrolumina-dev
```

Without the Doppler CLI, copy the 4 values from the dashboard and substitute
them for the `$..._TAG` variables above.

Expected: the Deployments restart on the exact tags from Doppler
(`kubectl describe deploy/frontend -n astrolumina-dev | grep Image:` shows
the tag). If a pod reports `ErrImagePull` with `manifest unknown`, the tag
in Doppler does not exist on GHCR — fix the value in Doppler and re-run.

## 4. Create the token Secret and let the operator sync

The token is deliberately NOT in git. Create it imperatively:

```bash
kubectl create secret generic doppler-token-dev -n doppler-operator-system \
  --from-literal=serviceToken='<paste-dev-service-token-here>'
```

The 4 `DopplerSecret` resources from `03-doppler-secrets.yaml` are already in
the cluster (applied in step 3). The operator now syncs the `dev` config.
Verify:

```bash
kubectl get dopplersecrets -n doppler-operator-system
kubectl describe dopplersecret astrolumina-dev-payment-api -n doppler-operator-system
```

Expected: no errors in `describe` output. Then confirm a real value landed
(prints only the first 8 chars, never paste full secrets in chat/logs):

```bash
kubectl get secret env-payment-api-secrets -n astrolumina-dev \
  -o jsonpath='{.data.STRIPE_SK}' | base64 -d | cut -c1-8
```

Expected: `sk_test_` or `sk_live_`, NOT the placeholder text. Repeat the
spirit of this check for one key per Secret if you want to be thorough.

## 5. Remove the placeholders

Now that Doppler owns the Secrets, stop applying the fake ones:

1. Edit `development/kustomization.yaml` and delete the
   `- 02-secrets.yaml` line.
2. Re-apply:

```bash
kubectl apply -k development/
kubectl rollout status deploy/frontend -n astrolumina-dev
kubectl get pods -n astrolumina-dev
```

Expected: pods restart automatically (the `secrets.doppler.com/reload`
annotation tells the operator to roll them) and come back `Running`.

## 6. Access from your host browser

Find a node IP (`kubectl get nodes -o wide`), then open:

- Frontend: `http://<node-ip>:30080`
- Astrology API: `http://<node-ip>:30301`
- Payment API: `http://<node-ip>:30302`
- Booking API: `http://<node-ip>:30303`

## 7. If something is wrong

- Pod stuck in `ImagePullBackOff` / `ErrImagePull` after step 3b: the
  `ghcr-secret` is missing, still the placeholder, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `kubectl get secret ghcr-secret -n astrolumina-dev` and
  `kubectl get events -n astrolumina-dev --sort-by=.lastTimestamp | tail -10`.
  Recreate the secret with the command from step 3b, then
  `kubectl rollout restart deploy -n astrolumina-dev`. (The old Compose
  equivalent was `echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER
  --password-stdin` — in K8s the kubelet needs the secret, not a node-local
docker login.)
- Pod `CrashLoopBackOff` with a "missing environment variable" log: the
  Doppler `dev` config lacks that key. Add it in Doppler, the operator syncs
  within seconds and restarts the pods by itself.
- `DopplerSecret` shows an error on `describe`: usually a wrong/expired
  service token or a wrong `project`/`config` name. Fix and re-apply, or
  recreate the token Secret (delete + `kubectl create secret ...` again).
- `kubectl get dopplersecrets -n doppler-operator-system` shows nothing:
  first check `kubectl get dopplersecrets -A` (maybe they landed in the app
  namespace — that means your checkout predates the kustomization namespace
  fix; pull latest and re-apply), then confirm the operator install from
  step 2 with `kubectl get crd dopplersecrets.secrets.doppler.com`.
- `kubectl get secrets` / `get pods` with no `-n` only shows the `default`
  namespace — always pass `-n astrolumina-dev` for this stack.
- NodePort unreachable from your machine: the VMs' firewall is the usual
  suspect. On a node, `ss -tlnp | grep 30080` must show a listener; if it
  does, open the port in the hypervisor/cloud firewall.
- HPA shows `<unknown>` metrics: expected, metrics-server is not bundled
  with RKE2. Pods still run fine; install metrics-server only if you want
  autoscaling during tests.
