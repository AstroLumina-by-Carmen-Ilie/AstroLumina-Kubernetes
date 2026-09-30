# Production — step-by-step deploy guide

Deploys into namespace `astrolumina-prod`: blue + green variants at
3 replicas each, Traefik `IngressRoute` on `web` (redirect to HTTPS) +
`websecure` (TLS via the `letsencrypt` certResolver), single host
`astrolumina.ro` with `/api/*` prefix routing, dashboard at
`dashboard.astrolumina.ro` protected by basicAuth. Blue is live by default.

Read this whole file once before running anything: steps 2 and 3 must happen
BEFORE the first `kubectl apply -k production/`.

## 0. Prerequisites (have these ready before you start)

- Fresh RKE2 VMs: 1 control-plane + 2 workers, all `Ready`.
- `kubectl` on your local machine plus the kubeconfig of the cluster.
- In Doppler: project `astrolumina`, config `prd` with LIVE values for
  EVERY variable (shared vars, frontend/API URLs with the `astrolumina.ro`
  host, `CORS_ORIGINS`, `sk_live_`, live price IDs, prod DSNs). It must
  also contain `GITHUB_USER` and `GITHUB_TOKEN` (PAT with `read:packages`)
  — the GHCR credentials used in step 5b. It must also contain the four image
  tags `FRONTEND_DOCKER_IMAGE_TAG` (2.0.6),
  `ASTROLOGY_API_DOCKER_IMAGE_TAG` (2.0.6), `BOOKING_API_DOCKER_IMAGE_TAG`
  (2.0.5), `PAYMENT_API_DOCKER_IMAGE_TAG` (2.0.6) — the exact tags pinned
  in step 5c. Plus the **service token** for
  the `prd` config.
- The images referenced by the Deployments are private on GHCR. Every
  blue/green Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  `01-ghcr-secret.yaml` ships as a placeholder; step 5b replaces it with
  the real secret built from the Doppler `GITHUB_USER` / `GITHUB_TOKEN`.

## 1. Point kubectl at the fresh cluster

```bash
export KUBECONFIG=/path/to/rke2.yaml
kubectl get nodes
```

Expected: 3 nodes, all `Ready`. If not, stop here and fix the VMs first.

## 2. Register the Let's Encrypt resolver in RKE2 Traefik (REQUIRED, first)

RKE2 installs Traefik from a HelmChart without any certResolver. The
production IngressRoute references a resolver named `letsencrypt`, so create
it before deploying anything. On the **control-plane node**, write
`/var/lib/rancher/rke2/server/manifests/traefik-config.yaml`:

```yaml
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
```

RKE2 picks the file up automatically and restarts Traefik. Verify the pod
came back and the resolver is known:

```bash
kubectl -n kube-system get pods | grep traefik
kubectl -n kube-system logs <traefik-pod-name> | grep -i "letsencrypt\|acme" | head -5
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
hash (needs the `htpasswd` binary; `apt install apache2-utils` on Debian):

```bash
htpasswd -nbB admin '<choose-a-strong-password>'
```

Copy the whole `admin:$2y$...` output line and paste it as the `users:` value
in `production/54-dashboard-auth-secret.yaml`, replacing
`REPLACE_WITH_HTPASSWD_LINE_FOR_DASHBOARD`. Do NOT commit the real hash if
this repo is shared; inject it at deploy time instead.

## 4. Install the Doppler operator (once per cluster rebuild)

```bash
kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml
kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s
kubectl get crd dopplersecrets.secrets.doppler.com
```

Expected: deployment `Available`, CRD exists.

## 5. Deploy production (placeholders first, real dashboard hash)

```bash
cd AstroLumina-Kubernetes
kubectl apply -k production/
kubectl get pods -n astrolumina-prod
```

Expected: 8 pods created. They will show `ImagePullBackOff` / `ErrImagePull`
until step 5b replaces the `ghcr-secret` placeholder — that is normal.
On `CrashLoopBackOff` (pulled fine, then crashed), inspect first:

```bash
kubectl logs -n astrolumina-prod deploy/frontend-blue --tail=30
```

## 5b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

`01-ghcr-secret.yaml` applied in step 5 is a placeholder with fake
credentials, so kubelet cannot pull the private GHCR images yet. The real
`GITHUB_USER` / `GITHUB_TOKEN` already live in Doppler (config `prd`) —
build the real secret imperatively (same pattern as `doppler-token-prd`;
it cannot be synced by the Doppler operator because a pull secret must be
`type: kubernetes.io/dockerconfigjson`, not `Opaque`):

```bash
kubectl create secret docker-registry ghcr-secret \
  -n astrolumina-prod \
  --docker-server=ghcr.io \
  --docker-username='<paste-GITHUB_USER-from-Doppler-prd>' \
  --docker-password='<paste-GITHUB_TOKEN-from-Doppler-prd>' \
  --docker-email='admin@astrolumina.com' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Or straight from the Doppler CLI:

```bash
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config prd)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config prd)
kubectl create secret docker-registry ghcr-secret -n astrolumina-prod \
  --docker-server=ghcr.io --docker-username="$GITHUB_USER" \
  --docker-password="$GITHUB_TOKEN" --docker-email='admin@astrolumina.com' \
  --dry-run=client -o yaml | kubectl apply -f -
unset GITHUB_TOKEN GITHUB_USER
```

Then force a re-pull and verify:

```bash
kubectl rollout restart deploy -n astrolumina-prod
kubectl get pods -n astrolumina-prod
kubectl get events -n astrolumina-prod --sort-by=.lastTimestamp | tail -10
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
applied imperatively here, same pattern as steps 5b and 6.) Pin both colors:

```bash
export FRONTEND_TAG=$(doppler secrets get FRONTEND_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
export ASTROLOGY_TAG=$(doppler secrets get ASTROLOGY_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
export BOOKING_TAG=$(doppler secrets get BOOKING_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
export PAYMENT_TAG=$(doppler secrets get PAYMENT_API_DOCKER_IMAGE_TAG --plain --project astrolumina --config prd)
for COLOR in blue green; do
  kubectl set image deploy/frontend-$COLOR frontend=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-frontend:$FRONTEND_TAG -n astrolumina-prod
  kubectl set image deploy/astrology-api-$COLOR astrology-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-astrologyapi:$ASTROLOGY_TAG -n astrolumina-prod
  kubectl set image deploy/booking-api-$COLOR booking-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-bookingapi:$BOOKING_TAG -n astrolumina-prod
  kubectl set image deploy/payment-api-$COLOR payment-api=ghcr.io/astrolumina-by-carmen-ilie/astrolumina-paymentapi:$PAYMENT_TAG -n astrolumina-prod
done
unset FRONTEND_TAG ASTROLOGY_TAG BOOKING_TAG PAYMENT_TAG
kubectl get pods -n astrolumina-prod
```

Without the Doppler CLI, copy the 4 values from the dashboard and substitute
them for the `$..._TAG` variables above.

Expected: all 8 Deployments restart on the exact tags from Doppler.
Double-check you read the `prd` config, not `stg` — deploying staging tags
to production is the classic blue-green footgun this step exists to prevent.
If a pod reports `ErrImagePull` with `manifest unknown`, the tag in Doppler
does not exist on GHCR — fix the value in Doppler and re-run. Run this again
on BOTH colors every time you promote a new build.

## 6. Create the token Secret and let the operator sync

```bash
kubectl create secret generic doppler-token-prd -n doppler-operator-system \
  --from-literal=serviceToken='<paste-production-service-token-here>'
```

Verify (prints only the first 8 chars of one key):

```bash
kubectl get dopplersecrets -n doppler-operator-system
kubectl describe dopplersecret astrolumina-production-payment-api -n doppler-operator-system
kubectl get secret env-payment-api-secrets -n astrolumina-prod \
  -o jsonpath='{.data.STRIPE_SK}' | base64 -d | cut -c1-8
```

Expected: no errors; the key starts with `sk_live_` (NOT `sk_test_`, NOT the
placeholder). Double-check you synced the `prd` config, not `stg`: charging
real money with test keys fails, and vice versa.

## 7. Remove the placeholders

1. Edit `production/kustomization.yaml` and delete the `- 02-secrets.yaml` line.
2. Re-apply and confirm the automatic restart:

```bash
kubectl apply -k production/
kubectl get pods -n astrolumina-prod
```

## 8. Reach it from your host

Get a node IP (`kubectl get nodes -o wide`) and add to `/etc/hosts`:

```text
192.168.1.10 astrolumina.ro dashboard.astrolumina.ro
```

- App: `https://astrolumina.ro`
- Dashboard: `https://dashboard.astrolumina.ro` (admin + password from step 3)
- API spot checks:

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

Same drill as staging (see `staging/README.md` step 7): verify green,
port-forward smoke test, flip the 4 `variant:` selectors in
`53-live-services.yaml`, re-apply, re-run the step 8 checks.

## 10. If something is wrong

- Pod stuck in `ImagePullBackOff` / `ErrImagePull` after step 5b: the
  `ghcr-secret` is missing, still the placeholder, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `kubectl get secret ghcr-secret -n astrolumina-prod` and
  `kubectl get events -n astrolumina-prod --sort-by=.lastTimestamp | tail -10`.
  Recreate the secret with the command from step 5b, then
  `kubectl rollout restart deploy -n astrolumina-prod`. (The old Compose
  equivalent was `echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER
  --password-stdin` — in K8s the kubelet needs the secret, not a node-local
docker login.)
- Browser cert warning on a supposedly public setup: LE never issued.
  Check Traefik logs for ACME errors (usually port 80 not publicly
  reachable, or wrong email/rate limits). See the IMPORTANT note in step 2.
- Dashboard asks for password forever / 401: the `users:` hash in
  `54-dashboard-auth-secret.yaml` is still the placeholder or was pasted
  with extra whitespace. Fix, re-apply, restart Traefik pod.
- Pod `CrashLoopBackOff` with missing env: add the key to the Doppler `prd`
  config; sync + restart are automatic.
- `kubectl get dopplersecrets -n doppler-operator-system` shows nothing:
  first check `kubectl get dopplersecrets -A` (maybe they landed in the app
  namespace — that means your checkout predates the kustomization namespace
  fix; pull latest and re-apply), then confirm the operator install from
  step 4 with `kubectl get crd dopplersecrets.secrets.doppler.com`.
- `kubectl get secrets` / `get pods` with no `-n` only shows the `default`
  namespace — always pass `-n astrolumina-prod` for this stack.
- HPA `<unknown>` metrics: install metrics-server if you want real
  autoscaling data; harmless otherwise.
