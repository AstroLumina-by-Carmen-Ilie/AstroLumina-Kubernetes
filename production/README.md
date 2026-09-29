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
  host, `CORS_ORIGINS`, `sk_live_`, live price IDs, prod DSNs), plus the
  **service token** for the `prd` config.
- The images referenced by the Deployments must be reachable from the nodes.

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

Expected: 8 pods, all `Running`. On failure, inspect first:

```bash
kubectl logs -n astrolumina-prod deploy/frontend-blue --tail=30
```

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

- Browser cert warning on a supposedly public setup: LE never issued.
  Check Traefik logs for ACME errors (usually port 80 not publicly
  reachable, or wrong email/rate limits). See the IMPORTANT note in step 2.
- Dashboard asks for password forever / 401: the `users:` hash in
  `54-dashboard-auth-secret.yaml` is still the placeholder or was pasted
  with extra whitespace. Fix, re-apply, restart Traefik pod.
- Pod `CrashLoopBackOff` with missing env: add the key to the Doppler `prd`
  config; sync + restart are automatic.
- HPA `<unknown>` metrics: install metrics-server if you want real
  autoscaling data; harmless otherwise.
