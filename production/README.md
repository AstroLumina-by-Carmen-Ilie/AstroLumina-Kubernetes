# Production — step-by-step deploy guide

Deploys into namespace `astrolumina-prod`: blue + green variants at
3 replicas each, Traefik `IngressRoute` on `web` (redirect to HTTPS) +
`websecure` (TLS via the `letsencrypt` certResolver), single host
`astrolumina.ro` with `/api/*` prefix routing, dashboard at
`dashboard.astrolumina.ro` protected by basicAuth. Blue is live by default.

Read this whole file once before running anything: steps 2 and 3 must happen
BEFORE the first `ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production"`.

## How this environment boots (read first)

Everything below assumes an **empty cluster** — no operator, no secrets,
nothing yet. The deploy works in three phases:

1. **Placeholders first.** `01-ghcr-secret.yaml` (fake pull credentials)
   and `02-secrets.yaml` (schema-exact keys, fake values) exist so the very
   first `kubectl apply -k` succeeds on a blank cluster: namespace,
   Deployments, Services and Secrets are all created and the pods start.
   They cannot pull images or read real config yet — `ImagePullBackOff`
   until step 5b is expected, not an error.
2. **Doppler fills in the real values.** The Doppler Kubernetes operator
   watches the `DopplerSecret` objects (`03-doppler-secrets.yaml`). Each one
   points at one Doppler config (here: `prd`) plus a service token, and
   declares which keys to sync into which Kubernetes Secret. Once the token
   Secret exists, the operator overwrites the placeholder values in place
   with the real ones and rolls the pods
   (`secrets.doppler.com/reload` annotation) — no redeploy needed.
   Two secrets cannot come from Doppler and are created imperatively
   instead: `ghcr-secret` (a pull secret must be
   `type: kubernetes.io/dockerconfigjson`, while the operator only syncs
   `Opaque`) and the token Secret itself (the operator needs the token to
   authenticate — chicken-and-egg).
3. **Placeholders leave the build.** After the sync is verified, the
   `- 01-ghcr-secret.yaml` and `- 02-secrets.yaml` lines are deleted from
   `kustomization.yaml` (step 7). If they stayed listed, the next
   `apply -k` would overwrite the live secrets with fakes again: `02`
   clobbers the Doppler-synced values, `01` clobbers the real pull secret
   and the following rollout dies with `ImagePullBackOff`. Git keeps the
   files — a fresh rebuild starts from placeholders again.

## 0. Prerequisites (have these ready before you start)

- Fresh RKE2 VMs: 1 control-plane + 2 workers, all `Ready`.
- Two-machine model: `doppler` CLI (logged in) runs on your LAPTOP — it only
  reads secrets into shell variables. `kubectl` runs on the CONTROL PLANE
  only, so every `kubectl` below is prefixed with
  `ssh "$K8S_CP_CONN" -- "..."`. This repo is mounted live at `/mnt/k8s`
  on the control plane — edits you make here apply straight from there, no
  `git pull` needed on the VM.
- In Doppler: project `astrolumina`, config `prd` with LIVE values for
  the complete runtime set (every key the Deployments reference — the
  manifests plus `02-secrets.yaml` are the exact schema, so no separate
  list is needed here). Three groups of keys take part in the setup
  itself, and you will export each of them below:
  - `GITHUB_USER`, `GITHUB_TOKEN` (PAT with `read:packages`) and
    `GITHUB_EMAIL` — the GHCR pull credentials used in step 5b
    (`dockerconfigjson` needs all three: username, password, email).
  - `FRONTEND_DOCKER_IMAGE_TAG` (2.0.6),
    `ASTROLOGY_API_DOCKER_IMAGE_TAG` (2.0.6),
    `BOOKING_API_DOCKER_IMAGE_TAG` (2.0.5),
    `PAYMENT_API_DOCKER_IMAGE_TAG` (2.0.6) — the exact tags pinned in
    step 5c.
  - `K8S_SERVICE_TOKEN` — the service token of the `prd` config,
    consumed in step 6. No dashboard copy-paste anywhere: every value
    below comes from `doppler secrets get`.
- The images referenced by the Deployments are private on GHCR. Every
  blue/green Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  `01-ghcr-secret.yaml` ships as a placeholder; step 5b replaces it with
  the real secret built from the Doppler `GITHUB_USER` / `GITHUB_TOKEN` /
  `GITHUB_EMAIL`.

## 1. Connect to the control plane (laptop → CP)

`K8S_CP_CONN` (e.g. `ubuntu@192.168.122.10`) already lives in Doppler
(config `prd`). Export it once per shell, then run `kubectl` on the CP
through it. Quoting rule for every `ssh` below: the remote command sits in
DOUBLE quotes, so `$VARS` from `doppler` expand on your laptop (intended —
the CP never sees Doppler), while `|` pipes inside the quotes still execute
on the CP:

```bash
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config prd)
ssh "$K8S_CP_CONN" -- "kubectl get nodes"
```

If `kubectl: command not found` (fresh VM — RKE2 hides its binary outside
the default PATH and ships no kubeconfig for your user), run this once per
control plane — afterwards plain `kubectl` works in every `ssh` below:

```bash
ssh "$K8S_CP_CONN" -- "sudo ln -sf /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl && mkdir -p ~/.kube && sudo cat /etc/rancher/rke2/rke2.yaml > ~/.kube/config && chmod 600 ~/.kube/config && kubectl get nodes"
```

Expected: 3 nodes, all `Ready`. If not, stop here and fix the VMs first.

## 2. Register the Let's Encrypt resolver in RKE2 Traefik (REQUIRED, first)

RKE2 installs Traefik from a HelmChart without any certResolver. The
production IngressRoute references a resolver named `letsencrypt`, so create
it before deploying anything — one `ssh` writing the file on the CP
(`tee` needs `sudo` for the RKE2 manifests dir; the `<<'EOF'` heredoc is
quoted so nothing expands anywhere by accident):

```bash
ssh "$K8S_CP_CONN" -- "sudo tee /var/lib/rancher/rke2/server/manifests/traefik-config.yaml > /dev/null <<'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-traefik
  namespace: kube-system
spec:
  valuesContent: |-
    persistence:
      enabled: true
    certResolvers:
      letsencrypt:
        email: admin@astrolumina.com
        storage: /data/acme.json
        httpChallenge:
          entryPoint: web
EOF"
```

RKE2 picks the file up automatically and restarts Traefik. Verify the pod
came back and the resolver is known:

```bash
ssh "$K8S_CP_CONN" -- "kubectl -n kube-system get pods | grep traefik"
ssh "$K8S_CP_CONN" -- "kubectl -n kube-system logs deploy/rke2-traefik | grep -i letsencrypt | head -5"
```

IMPORTANT, read twice: Let's Encrypt `httpChallenge` requires the public
internet to reach your domain on port 80. This works only if
`astrolumina.ro` resolves publicly to your node IP. If you are testing
locally with a fake `/etc/hosts` entry (step 7), issuance WILL fail and
Traefik will serve its default self-signed cert instead (browser warning,
`curl -k` needed). That is fine for a connectivity test, but do NOT mistake
it for a working certificate setup. For a real certificate, point real DNS
at the node and re-apply.

## 3. Set the dashboard password (REQUIRED, before first apply)

`54-dashboard-auth-secret.yaml` ships with a placeholder. Generate the real
hash on your LAPTOP (needs the `htpasswd` binary; `apt install apache2-utils`
on Debian), then edit the local file — the CP sees it live at `/mnt/k8s`:

```bash
htpasswd -nbB admin '<choose-a-strong-password>'
```

Copy the whole `admin:$2y$...` output line and paste it as the `users:` value
in `production/54-dashboard-auth-secret.yaml`, replacing
`REPLACE_WITH_HTPASSWD_LINE_FOR_DASHBOARD`. Do NOT commit the real hash if
this repo is shared; inject it at deploy time instead.

## 4. Install the Doppler operator (once per cluster rebuild)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml"
ssh "$K8S_CP_CONN" -- "kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s"
ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"
```

Expected: deployment `Available`, CRD exists.

## 5. Deploy production (placeholders first, real dashboard hash)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-prod"
```

Expected: 8 pods created. They will show `ImagePullBackOff` / `ErrImagePull`
until step 5b replaces the `ghcr-secret` placeholder — that is normal.
On `CrashLoopBackOff` (pulled fine, then crashed), inspect first:

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -n astrolumina-prod deploy/frontend-blue --tail=30"
```

## 5b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

`01-ghcr-secret.yaml` applied in step 5 is a placeholder with fake
credentials, so kubelet cannot pull the private GHCR images yet. The real
`GITHUB_USER` / `GITHUB_TOKEN` / `GITHUB_EMAIL` already live in Doppler
(config `prd`) — build the real secret imperatively from them, zero
copy-paste (same pattern as `doppler-token-prd`; it cannot be synced by the
Doppler operator because a pull secret must be
`type: kubernetes.io/dockerconfigjson`, not `Opaque`):

The three `export` lines run on your LAPTOP (Doppler lives here). The
`kubectl create` runs on the CP — note the quote splice
`'...'"$VAR"'...'`: secret values expand locally, so tokens with `$`,
`!` or quotes survive the trip intact, while the `|` pipe still executes
remotely:

```bash
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config prd)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config prd)
export GITHUB_EMAIL=$(doppler secrets get GITHUB_EMAIL --plain --project astrolumina --config prd)
ssh "$K8S_CP_CONN" -- 'kubectl create secret docker-registry ghcr-secret -n astrolumina-prod --docker-server=ghcr.io --docker-username='"$GITHUB_USER"' --docker-password='"$GITHUB_TOKEN"' --docker-email='"$GITHUB_EMAIL"' --dry-run=client -o yaml | kubectl apply -f -'
unset GITHUB_TOKEN GITHUB_USER GITHUB_EMAIL
```

Then force a re-pull and verify:

```bash
ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-prod"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-prod"
ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-prod --sort-by=.lastTimestamp | tail -10"
```

Expected: pods leave `ImagePullBackOff` and reach `Running` (they still boot
with the `02-secrets.yaml` placeholder env values until step 6 syncs Doppler).
Do NOT commit the real credentials — `01-ghcr-secret.yaml` stays a placeholder
in git.

## 5c. Pin the images to the Doppler tags (blue AND green)

The manifests ship every image as `:latest`. The real tag per service lives
in Doppler (config `prd`): `FRONTEND_DOCKER_IMAGE_TAG` (2.0.6),
`ASTROLOGY_API_DOCKER_IMAGE_TAG` (2.0.6), `BOOKING_API_DOCKER_IMAGE_TAG`
(2.0.5), `PAYMENT_API_DOCKER_IMAGE_TAG` (2.0.6). (A Deployment's `image:`
field is static — Kubernetes cannot read it from a Secret — so the tags are
applied imperatively here, same pattern as steps 5b and 6.) `export` lines
on the laptop, `set image` on the CP (tags are plain version strings,
double quotes are safe). Pin both colors:

```bash
export FRONTEND_TAG=$(doppler secrets get FRONTEND_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
export ASTROLOGY_TAG=$(doppler secrets get ASTROLOGY_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
export BOOKING_TAG=$(doppler secrets get BOOKING_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
export PAYMENT_TAG=$(doppler secrets get PAYMENT_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
for COLOR in blue green; do
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/frontend-$COLOR frontend=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-frontend:$FRONTEND_TAG -n astrolumina-prod"
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/astrology-api-$COLOR astrology-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-astrologyapi:$ASTROLOGY_TAG -n astrolumina-prod"
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/booking-api-$COLOR booking-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-bookingapi:$BOOKING_TAG -n astrolumina-prod"
  ssh "$K8S_CP_CONN" -- "kubectl set image deploy/payment-api-$COLOR payment-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-paymentapi:$PAYMENT_TAG -n astrolumina-prod"
done
unset FRONTEND_TAG ASTROLOGY_TAG BOOKING_TAG PAYMENT_TAG
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-prod"
```

Expected: all 8 Deployments restart on the exact tags from Doppler.
Double-check you read the `prd` config, not `stg` — deploying staging tags
to production is the classic blue-green footgun this step exists to prevent.
If a pod reports `ErrImagePull` with `manifest unknown`, the tag in Doppler
does not exist on GHCR — fix the value in Doppler and re-run. Run this again
on BOTH colors every time you promote a new build.

## 6. Create the token Secret and let the operator sync

The token is deliberately NOT in git. It already lives in Doppler as
`K8S_SERVICE_TOKEN` (per-environment value, `prd` config) — pull it into
your shell, then consume it (idempotent, safe to re-run):

```bash
export K8S_SERVICE_TOKEN=$(doppler secrets get K8S_SERVICE_TOKEN --plain --project astrolumina --config prd)
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic doppler-token-prd -n doppler-operator-system --from-literal=serviceToken='"$K8S_SERVICE_TOKEN"' --dry-run=client -o yaml | kubectl apply -f -'
```

If `doppler secrets get K8S_SERVICE_TOKEN` ever reports the key missing
(e.g. revoked upstream and never re-saved), mint a fresh one without
touching the dashboard (uses your existing Doppler CLI login) and store it
back in Doppler under the same name, so this step stays zero-paste:

```bash
export K8S_SERVICE_TOKEN=$(doppler configs tokens create k8s-prd --project astrolumina --config prd --plain)
```

then re-run the `kubectl create secret` above. (Generated tokens pile up in
Doppler — revoke the ones you no longer use:
`doppler configs tokens revoke <token-id> --project astrolumina --config prd`.)

Verify (prints only the first 8 chars of one key):

```bash
ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"
ssh "$K8S_CP_CONN" -- "kubectl describe dopplersecret astrolumina-production-payment-api -n doppler-operator-system"
ssh "$K8S_CP_CONN" -- "kubectl get secret env-payment-api-secrets -n astrolumina-prod -o jsonpath='{.data.STRIPE_SK}' | base64 -d | cut -c1-8"
```

Expected: no errors; the key starts with `sk_live_` (NOT `sk_test_`, NOT the
placeholder). Double-check you synced the `prd` config, not `stg`: charging
real money with test keys fails, and vice versa.

## 7. Remove the placeholders (REQUIRED — BOTH files)

As long as the placeholder files stay listed in `kustomization.yaml`, every
future `apply -k` overwrites the REAL secrets with fakes: `02-secrets.yaml`
would clobber the Doppler-synced values, and `01-ghcr-secret.yaml` would
clobber the real pull secret — so the next rollout restart dies with
`ImagePullBackOff`. Doppler already synced without any redeploy; this step
just stops kustomize from ever stomping on the live secrets again.

Edit `production/kustomization.yaml` and delete BOTH lines:
`- 02-secrets.yaml` AND `- 01-ghcr-secret.yaml`.
(Git keeps the originals — a fresh rebuild starts from placeholders again.)
Then re-apply:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-prod"
```

## 8. Reach it from your host

Get a node IP (`ssh "$K8S_CP_CONN" -- "kubectl get nodes -o wide"`) and add to `/etc/hosts` on your laptop:

```text
192.168.1.10 astrolumina.ro dashboard.astrolumina.ro
```

- App: `https://astrolumina.ro`
- Dashboard: `https://dashboard.astrolumina.ro` (admin + password from step 3)
- API spot checks (from your LAPTOP — they test your host → node path):

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://astrolumina.ro/api/astrology/<health-path>
curl -sk -o /dev/null -w '%{http_code}\n' https://astrolumina.ro/api/booking/<health-path>
curl -sk -o /dev/null -w '%{http_code}\n' https://astrolumina.ro/api/payment/<health-path>
```

(`-k` only while the certificate is not publicly trusted; drop it once LE
issued a real cert, otherwise you are not testing TLS.) With a real cert,
confirm it explicitly:

```bash
echo | openssl s_client -connect astrolumina.ro:443 -servername astrolumina.ro 2>/dev/null | grep -i "issuer\|verify return"
```

Expected issuer: Let's Encrypt.

## 9. Blue-green cutover

Same drill as staging (see `staging/README.md` step 7): verify green
(`ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-prod -l variant=green"`),
port-forward smoke test via `ssh -L`, flip the 4 `variant:` selectors in
`53-live-services.yaml`, re-apply (`ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production"`),
re-run the step 8 checks.

## 10. If something is wrong

- Pod stuck in `ImagePullBackOff` / `ErrImagePull` after step 5b: the
  `ghcr-secret` is missing, still the placeholder, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-prod"`
  and `ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-prod --sort-by=.lastTimestamp | tail -10"`.
  Recreate the secret with the command from step 5b, then
  `ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-prod"`.
- Browser cert warning on a supposedly public setup: LE never issued.
  Check Traefik logs for ACME errors (usually port 80 not publicly
  reachable, or wrong email/rate limits). See the IMPORTANT note in step 2.
- Dashboard asks for password forever / 401: the `users:` hash in
  `54-dashboard-auth-secret.yaml` is still the placeholder or was pasted
  with extra whitespace. Fix, re-apply, restart Traefik pod.
- Pod `CrashLoopBackOff` with missing env: add the key to the Doppler `prd`
  config; sync + restart are automatic.
- `ssh "$K8S_CP_CONN" -- "kubectl get dopplersecrets -n doppler-operator-system"`
  shows nothing: the operator install from step 4 did not complete
  (confirm with `ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"`),
  or the token Secret is missing or misnamed — the DopplerSecrets must land
  in `doppler-operator-system` (re-run the check with `-A` to see which
  namespace yours went to).
- `kubectl get secrets` / `get pods` with no `-n` only shows the `default`
  namespace — always pass `-n astrolumina-prod` for this stack.
- HPA `<unknown>` metrics: install metrics-server if you want real
  autoscaling data; harmless otherwise.
