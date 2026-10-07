# Production — step-by-step deploy guide

Deploys into namespace `astrolumina-prod`: blue + green variants at
3 replicas each, Traefik `IngressRoute` HTTPS-only on `websecure`
(TLS via the `letsencrypt` certResolver), single host
`production.k8s.astrolumina.ro` with `/api/*` prefix routing, plus the
Traefik dashboard at `dashboard.k8s.astrolumina.ro` on HTTPS with basicAuth.
HTTP is not served. Blue is live by default.

Read this whole file once before running anything: steps 2 and 3 must happen
BEFORE the first `ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production"`.

## How this environment boots (read first)

Everything below assumes an **empty cluster** — no operator, no secrets,
nothing yet. The deploy works in two phases:

1. **Kustomize creates everything except secrets.** The first
   `kubectl apply -k` creates the namespace, Deployments, Services and
   `DopplerSecret` objects — but NO `ghcr-secret` and NO `env-*` Secrets
   (nothing in git provides them). The pods are created and wait:
   `ImagePullBackOff` until step 5b creates the real pull secret, and
   `CreateContainerConfigError` / `Pending` until step 6 gives the operator
   its token. Both states are expected, not errors — kubelet retries on
   its own and the pods recover by themselves once the secrets exist.
2. **Doppler fills in the real values.** The Doppler Kubernetes operator
   watches the `DopplerSecret` objects (`01-doppler-secrets.yaml`). Each one
   points at one Doppler config (here: `prd`) plus a service token, and
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

- Fresh RKE2 VMs: 1 control-plane + 2 workers, all `Ready`.
- Two-machine model: `doppler` CLI (logged in) runs on your LAPTOP — it only
  reads secrets into shell variables. `kubectl` runs on the CONTROL PLANE
  only, so every `kubectl` below is prefixed with
  `ssh "$K8S_CP_CONN" -- "..."`. This repo is mounted live at `/mnt/k8s`
  on the control plane — edits you make here apply straight from there, no
  `git pull` needed on the VM.
- In Doppler: project `astrolumina`, config `prd` with LIVE values for
  the complete runtime set (every key the Deployments reference — the
  manifests plus the `01-doppler-secrets.yaml` sync lists are the exact
  schema, so no separate list is needed here). Two groups of keys take part in the setup
  itself, and you will export each of them below:
  - `GITHUB_USER`, `GITHUB_TOKEN` (PAT with `read:packages`) and
    `GITHUB_EMAIL` — the GHCR pull credentials used in step 5b
    (`dockerconfigjson` needs all three: username, password, email).
  - `DOPPLER_SERVICE_TOKEN` — the service token of the `prd` config,
    consumed in step 6. No dashboard copy-paste anywhere: every value
    below comes from `doppler secrets get`.
- The images referenced by the Deployments are private on GHCR. Every
  blue/green Deployment references `imagePullSecrets: [{name: ghcr-secret}]`.
  No pull secret ships in git; step 5b creates the real one from the
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
nothing lands on the CP. Every Deployment runs 3 replicas and each HPA
scales its app 3–5. At max burst blue still fits worker-01 (~1600m CPU /
~2.1Gi of 2000m/2908Mi including system load); anything beyond stays
Pending — lab tradeoff, the live color keeps serving.

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
`production.k8s.astrolumina.ro` resolves publicly to your node IP. If you are testing
locally with a fake `/etc/hosts` entry (step 7), issuance WILL fail and
Traefik will serve its default self-signed cert instead (browser warning,
`curl -k` needed). That is fine for a connectivity test, but do NOT mistake
it for a working certificate setup. For a real certificate, point real DNS
at the node and re-apply.

## 3. Create the namespace + dashboard password Secret (REQUIRED, before first apply)

There is no dashboard password file in git at all — the Secret is created
purely imperatively (like `ghcr-secret`). The real hash already lives in
Doppler as
`TRAEFIK_DASHBOARD_AUTH` (`prd` config, full `admin:$2y$...` line). The
`astrolumina-prod` namespace does not exist yet at this point (it is normally
created by `apply -k` in step 5), so create it explicitly first — re-applying
it later via `apply -k` is harmless:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f /mnt/k8s/production/00-namespace.yaml"
export TRAEFIK_DASHBOARD_AUTH=$(doppler secrets get TRAEFIK_DASHBOARD_AUTH --plain --project astrolumina --config prd)
ESCAPED_AUTH=${TRAEFIK_DASHBOARD_AUTH//\$/\\\$}
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic traefik-dashboard-auth -n astrolumina-prod --from-literal=users='"$ESCAPED_AUTH"' --dry-run=client -o yaml | kubectl apply -f -'
unset ESCAPED_AUTH TRAEFIK_DASHBOARD_AUTH
```

## 4. Install the Doppler operator (once per cluster rebuild)

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml"
ssh "$K8S_CP_CONN" -- "kubectl wait --for=condition=Available deploy -n doppler-operator-system --all --timeout=180s"
ssh "$K8S_CP_CONN" -- "kubectl get crd dopplersecrets.secrets.doppler.com"
```

Expected: deployment `Available`, CRD exists.

## 5. Deploy production

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-prod"
```

Expected: 8 pods created. They will show `ImagePullBackOff` / `ErrImagePull`
until step 5b creates the real `ghcr-secret` — that is normal. They will
also sit in `CreateContainerConfigError` / `Pending` until step 6, because
no `env-*` Secrets exist yet — also normal: kubelet retries on its own
and the pods start by themselves once Doppler syncs.
On `CrashLoopBackOff` (pulled fine, then crashed), inspect first:

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -n astrolumina-prod deploy/frontend-blue --tail=30"
```

## 5b. Create the GHCR pull secret (REQUIRED, once per cluster rebuild)

No pull secret exists yet (nothing in git provides one), so kubelet cannot
pull the private GHCR images yet. The real
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

Expected: pods leave `ImagePullBackOff` and reach `Running` (config still
missing until step 6 syncs Doppler — the `CreateContainerConfigError` note
in step 5 applies).
Do NOT commit the real credentials — create them only with the command
above, never in a file.

## 5c. Image tags are pinned in git (blue AND green)

Image tags are the source of truth in git — the same model as the Compose
`versions.env` files (the production `versions.env` holds the same values, so
keep them in parity). Doppler no longer carries any `*_DOCKER_IMAGE_TAG`
keys. Each Deployment's `image:` field already references its pinned tag
(frontend `2.0.8`, astrology-api `2.0.8`, booking-api `2.0.7`, payment-api
`2.0.8`), so applying the manifests deploys those exact tags on both colors
— there is nothing else to run.

To promote a new build, bump the tag in the blue AND green Deployment files
(keep both colors identical unless you are staging a release on the idle
color), commit, and re-apply:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production && kubectl get pods -n astrolumina-prod"
```

Expected: all 8 Deployments run the tags from git
(`ssh "$K8S_CP_CONN" -- "kubectl describe deploy/frontend-blue -n astrolumina-prod | grep Image:"`
shows `:2.0.8`). If a pod reports `ErrImagePull` with `manifest unknown`,
the tag does not exist on GHCR — fix the value in the manifests and re-apply.
Double-check parity with the production Compose `versions.env`, and make sure
you read the `prd` values, not `stg` — deploying staging tags to production
is the classic blue-green footgun this step exists to prevent.

## 6. Create the token Secret and let the operator sync

The token is deliberately NOT in git. It already lives in Doppler as
`DOPPLER_SERVICE_TOKEN` (per-environment value, `prd` config) — pull it into
your shell, then consume it (idempotent, safe to re-run):

```bash
export DOPPLER_SERVICE_TOKEN=$(doppler secrets get DOPPLER_SERVICE_TOKEN --plain --project astrolumina --config prd)
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic doppler-token-prd -n doppler-operator-system --from-literal=serviceToken='"$DOPPLER_SERVICE_TOKEN"' --dry-run=client -o yaml | kubectl apply -f -'
```

If `doppler secrets get DOPPLER_SERVICE_TOKEN` ever reports the key missing
(e.g. revoked upstream and never re-saved), mint a fresh one without
touching the dashboard (uses your existing Doppler CLI login) and store it
back in Doppler under the same name, so this step stays zero-paste:

```bash
export DOPPLER_SERVICE_TOKEN=$(doppler configs tokens create k8s-prd --project astrolumina --config prd --plain)
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

Expected: no errors; the key starts with `sk_live_` (NOT `sk_test_`). Double-check you synced the `prd` config, not `stg`: charging
real money with test keys fails, and vice versa.

## 7. Prove the build is sync-safe (nothing to remove)

In the old layout this step deleted placeholder lines from
`kustomization.yaml`. That logic is gone: `01`/`02` are no longer listed,
so re-applying is a pure no-op on secrets — Doppler keeps owning the
values and ArgoCD can sync freely. Prove it:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -k /mnt/k8s/production"
ssh "$K8S_CP_CONN" -- "kubectl get pods -n astrolumina-prod"
```

## 8. Reach it from your host

Get a node IP (`ssh "$K8S_CP_CONN" -- "kubectl get nodes -o wide"`) and add to `/etc/hosts` on your laptop:

```text
192.168.122.11 production.k8s.astrolumina.ro dashboard.k8s.astrolumina.ro
```

- App: `https://production.k8s.astrolumina.ro`
- Dashboard: `https://dashboard.k8s.astrolumina.ro` (TLS + basicAuth, admin + password from step 3; plain `http://` serves the staging dashboard)
- API spot checks (from your LAPTOP — they test your host → node path):

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://production.k8s.astrolumina.ro/api/astrology/<health-path>
curl -sk -o /dev/null -w '%{http_code}\n' https://production.k8s.astrolumina.ro/api/booking/<health-path>
curl -sk -o /dev/null -w '%{http_code}\n' https://production.k8s.astrolumina.ro/api/payment/<health-path>
```

(`-k` only while the certificate is not publicly trusted; drop it once LE
issued a real cert, otherwise you are not testing TLS.) With a real cert,
confirm it explicitly:

```bash
echo | openssl s_client -connect production.k8s.astrolumina.ro:443 -servername production.k8s.astrolumina.ro 2>/dev/null | grep -i "issuer\|verify return"
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
  `ghcr-secret` is missing, or the Doppler
  `GITHUB_TOKEN` is expired / lacks `read:packages`. Check with
  `ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-prod"`
  and `ssh "$K8S_CP_CONN" -- "kubectl get events -n astrolumina-prod --sort-by=.lastTimestamp | tail -10"`.
  Recreate the secret with the command from step 5b, then
  `ssh "$K8S_CP_CONN" -- "kubectl rollout restart deploy -n astrolumina-prod"`.
- Browser cert warning on a supposedly public setup: LE never issued.
  Check Traefik logs for ACME errors (usually port 80 not publicly
  reachable, or wrong email/rate limits). See the IMPORTANT note in step 2.
- Dashboard asks for password forever / 401: the live `users:` value was
  created from a $-mangled hash (unquoted `$2`/`$05`/… expansion somewhere
  along the way). Re-run step 3 exactly as written — including the
  `ESCAPED_AUTH` line — then restart the Traefik pod.
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
