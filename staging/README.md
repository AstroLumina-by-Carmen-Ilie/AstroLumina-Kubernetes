# Staging — step-by-step deploy guide

Deploys into namespace `astrolumina-staging`: blue + green variants at
3 replicas each, Traefik `IngressRoute` on entryPoint `web` (plain HTTP),
single host `staging.astrolumina.ro` with `/api/*` prefix routing (mirrors
the Compose staging `routes.yml`). Blue is live by default.

## 0. Prerequisites (have these ready before you start)

- Fresh RKE2 VMs: 1 control-plane + 2 workers, all `Ready` (you rebuild the
  machines between environments).
- `kubectl` on your local machine plus the kubeconfig of the cluster.
- In Doppler: project `astrolumina`, config `stg`, filled with EVERY
  variable the stack needs (shared vars, frontend/API URLs with the
  `staging.astrolumina.ro` host, `CORS_ORIGINS`, plus all secrets), plus the
  **service token** for the `stg` config.
- The images referenced by the Deployments must be reachable from the nodes.

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

Expected: deployment `Available`, CRD exists.

## 3. Deploy staging (placeholders first)

```bash
cd AstroLumina-Kubernetes
kubectl apply -k staging/
kubectl get pods -n astrolumina-staging
```

Expected: 8 pods (`*-blue` and `*-green`), all `Running`. If any pod is
`CrashLoopBackOff`, inspect before continuing:

```bash
kubectl logs -n astrolumina-staging deploy/frontend-blue --tail=30
```

## 4. Create the token Secret and let the operator sync

```bash
kubectl create secret generic doppler-token-stg -n doppler-operator-system \
  --from-literal=serviceToken='<paste-staging-service-token-here>'
```

Verify the sync (prints only the first 8 chars of one key):

```bash
kubectl get dopplersecrets -n doppler-operator-system
kubectl describe dopplersecret astrolumina-staging-payment-api -n doppler-operator-system
kubectl get secret env-payment-api-secrets -n astrolumina-staging \
  -o jsonpath='{.data.STRIPE_SK}' | base64 -d | cut -c1-8
```

Expected: no errors on `describe`; the key starts with `sk_test_` (staging
uses test keys), not the placeholder text.

## 5. Remove the placeholders

1. Edit `staging/kustomization.yaml` and delete the `- 02-secrets.yaml` line.
2. Re-apply and confirm the automatic restart:

```bash
kubectl apply -k staging/
kubectl get pods -n astrolumina-staging
```

Expected: all 8 pods restart (via the `secrets.doppler.com/reload`
annotation) and return to `Running`.

## 6. Reach it from your host

Traefik gets a node IP from the bundled Klipper ServiceLB. Get one:

```bash
kubectl get nodes -o wide
```

Add to `/etc/hosts` on your local machine (replace with the real node IP):

```text
192.168.1.10 staging.astrolumina.ro
```

Then open `http://staging.astrolumina.ro` in the browser, and check the API
routes directly:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://staging.astrolumina.ro/api/astrology/<health-path>
curl -s -o /dev/null -w '%{http_code}\n' http://staging.astrolumina.ro/api/booking/<health-path>
curl -s -o /dev/null -w '%{http_code}\n' http://staging.astrolumina.ro/api/payment/<health-path>
```

(Replace `<health-path>` with each API's real health endpoint.) The
`/api/*` prefix is stripped before reaching the pods, exactly like Compose.

## 7. Blue-green cutover practice (staging is where you rehearse this)

Blue serves traffic now. To switch to green:

1. Confirm green is healthy: `kubectl get pods -n astrolumina-staging -l variant=green`.
2. Smoke-test green directly (bypass Traefik):
   `kubectl port-forward -n astrolumina-staging svc/frontend-green 8080:80`,
   then open `http://localhost:8080`.
3. Flip the 4 `selector: variant: blue` fields to `green` in
   `53-live-services.yaml` and `kubectl apply -k staging/`.
4. Re-run the step 6 curl checks. Roll back by flipping back to `blue`.

Keep the idle color running until the new one proves healthy.

## 8. If something is wrong

- `curl` returns 404 on `/api/*`: the request never matched the IngressRoute.
  Check `kubectl get ingressroute -n astrolumina-staging` and that your
  `/etc/hosts` points at a node IP (not the VM hostname).
- Pod `CrashLoopBackOff` with missing env: add the key to the Doppler `stg`
  config; sync + restart are automatic.
- `describe dopplersecret` shows auth errors: wrong token or wrong
  `project`/`config` in `03-doppler-secrets.yaml`.
- HPA `<unknown>` metrics: metrics-server is not bundled with RKE2; harmless
  for testing.
