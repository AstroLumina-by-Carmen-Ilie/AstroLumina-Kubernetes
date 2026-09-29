# Development — step-by-step deploy guide

Deploys the whole stack into namespace `astrolumina-dev`: 1 replica per
service, NodePort Services, no ingress (mirrors the Compose dev setup where
ports are published and no Traefik exists).

NodePorts: frontend `30080`, astrology `30301`, payment `30302`,
booking `30303`.

## 0. Prerequisites (have these ready before you start)

- RKE2 VMs up: 1 control-plane + 2 workers, all `Ready`.
- `kubectl` on your local machine plus the kubeconfig of the cluster.
- In Doppler: project `astrolumina`, config `dev`, filled with every secret
  key (see step 4 for the exact list), plus the **service token** for the
  `dev` config (Doppler dashboard -> project -> `dev` -> Access).
- The images referenced by the Deployments must be reachable from the nodes
  (registry credentials / public registry).

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

Expected: 4 pods, all `Running` (they boot with placeholder secrets; the
apps start because every referenced key exists, even if values are fake).
If a pod is `CrashLoopBackOff`, check why before continuing:

```bash
kubectl logs -n astrolumina-dev deploy/frontend --tail=30
```

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

- Pod `CrashLoopBackOff` with a "missing environment variable" log: the
  Doppler `dev` config lacks that key. Add it in Doppler, the operator syncs
  within seconds and restarts the pods by itself.
- `DopplerSecret` shows an error on `describe`: usually a wrong/expired
  service token or a wrong `project`/`config` name. Fix and re-apply, or
  recreate the token Secret (delete + `kubectl create secret ...` again).
- NodePort unreachable from your machine: the VMs' firewall is the usual
  suspect. On a node, `ss -tlnp | grep 30080` must show a listener; if it
  does, open the port in the hypervisor/cloud firewall.
- HPA shows `<unknown>` metrics: expected, metrics-server is not bundled
  with RKE2. Pods still run fine; install metrics-server only if you want
  autoscaling during tests.
