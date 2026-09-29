# AstroLumina-Kubernetes

Kubernetes manifests for AstroLumina, mirroring the logic of
AstroLumina-DockerCompose across the same three environments:
`development`, `staging`, `production`.

## Layout

```text
development/          # single replica set, NodePort services, no ingress
staging/
  blue/               # staging variant blue (replicas: 3)
  green/              # staging variant green (mirror of blue)
  51-middlewares.yaml # Traefik Middleware CRDs (mirror of Compose routes.yml)
  52-ingressroute.yaml# single-host IngressRoute on entryPoint web (HTTP)
  53-live-services.yaml # 4 stable *-live services = the traffic switch
production/           # same shape as staging + TLS + dashboard auth
03-doppler-secrets.yaml in each env # 4 DopplerSecrets (token made imperatively)
README.md in each env # copy-paste runbook: exact commands in order
```

Start with the runbook of the environment you deploy:
`development/README.md`, `staging/README.md`, `production/README.md`.
Each one lists prerequisites, every command in order, what output to expect,
and what to check when a step fails.

## Environment matrix

| Aspect | development | staging | production |
|---|---|---|---|
| Replicas | 1 | 3 | 3 |
| Exposure | NodePort (30080, 30301/30302/30303), no ingress | Traefik IngressRoute, entryPoint `web` (HTTP) | Traefik IngressRoute, `web` redirect + `websecure` (TLS) |
| Host | node IP | `staging.astrolumina.ro` | `astrolumina.ro` |
| API routing | direct NodePort per service | single host + `/api/*` path prefix, prefix stripped | same as staging |
| TLS | none | none | Let's Encrypt via `letsencrypt` certResolver |
| Dashboard | none | Traefik dashboard route, no auth | dashboard route + basicAuth (`dashboard.astrolumina.ro`) |
| HPA | min 1 / max 3 | min 3 / max 6 | min 3 / max 9 |

## Docker Compose parity

- **Single-host `/api/*` routing**: staging/production route
  `Host(host) && PathPrefix(/api/astrology|payment|booking)` to the
  matching `-live` Service with a strip-prefix middleware, exactly like the
  `routes.yml` dynamic config in Compose.
- **Middlewares**: `security-headers`, `compress`, `rate-limit`
  (100 burst / 50 average / 1s, defined but attached to no router — same as
  Compose), `strip-api-{astrology,payment,booking}`. Production adds
  `to-https` (redirectScheme) and `dashboard-auth` (basicAuth).
- **The 4-line switch**: live traffic is the 4 `selector: variant:` fields in
  `53-live-services.yaml`. Flip `blue` to `green` and re-apply to cut over.
- **Frontend env injection**: the frontend image reads `public/env.js`
  placeholders filled by Compose-style variables (`ASTROLOGICAL_API_URL`,
  `STRIPE_PK`, `*_SERVER_DNS/PORT`, `FRONTEND_SENTRY_DSN`, `R2_BASE_URL`),
  not `VITE_*`. Those variables live in Doppler and are synced into the
  `env-frontend-secrets` Secret, exactly like every other variable.
- **Backend env contract**: the APIs validate their full env set at startup
  (zod, exit 1 if missing), so every Deployment wires all 9 shared vars
  (`NODE_ENV`, `*_SERVER_PORT/DNS` x4) plus its service keys — all via
  `secretKeyRef` into the service's Doppler-managed Secret. There are no
  ConfigMaps left: 100% of variables come from Doppler.

## Secrets and Doppler

- `02-secrets.yaml` files contain **placeholders only** (schema-exact keys).
  Real secrets are never committed.
- The Doppler Kubernetes Operator (installed once per cluster rebuild) syncs
  one Doppler config into 4 Kubernetes Secrets per environment:

```bash
kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml
```

| Doppler config | Token Secret (imperative, NOT in git) | Namespace | Managed Secrets |
|---|---|---|---|
| `dev` | `doppler-token-dev` | `astrolumina-dev` | `env-frontend-secrets`, `env-astrology-api-secrets`, `env-booking-api-secrets`, `env-payment-api-secrets` |
| `stg` | `doppler-token-stg` | `astrolumina-staging` | same 4 names |
| `prd` | `doppler-token-prd` | `astrolumina-prod` | same 4 names |

- `03-doppler-secrets.yaml` in each environment holds the 4 `DopplerSecret`
  resources (one per managed Secret, key subsets included), so
  `kubectl apply -k <env>/` brings up everything at once. Create the token
  after each rebuild, e.g. for development:
  `kubectl create secret generic doppler-token-dev -n doppler-operator-system --from-literal=serviceToken='<token>'`.
- Boot order per environment: apply kustomize (placeholders) -> create token
  -> verify sync -> delete the `- 02-secrets.yaml` line from that
  environment's `kustomization.yaml` -> re-apply. Full command sequences with
  expected outputs live in each environment's `README.md`.
- Doppler must provide literally every variable the Deployments reference
  via `secretKeyRef` — shared vars (`NODE_ENV`, `*_SERVER_PORT/DNS`,
  `CORS_ORIGINS`), frontend runtime URLs, per-service URLs, plus all secrets
  (Sentry DSNs, API keys, Stripe keys, price IDs, R2/D1 credentials). If a
  single key is missing from the Doppler config, the pod fails at startup
  (zod validation, exit 1) and the fix is always "add the key in Doppler".
  Deployments carry the `secrets.doppler.com/reload` annotation, so pods
  restart automatically whenever a synced value changes.

## Blue-green runbook (staging / production)

1. Deploy to the idle color first:
   `kubectl apply -k staging/` (or verify only green changed).
2. Check pods and probes: `kubectl get pods -n astrolumina-staging`.
3. Smoke-test the idle color via port-forward on its direct Service.
4. Flip the 4 `variant:` selectors in `53-live-services.yaml`
   (`blue` <-> `green`) and re-apply.
5. Keep the previous color running until the new one proves healthy.

## RKE2 networking (1 control-plane + 2 workers)

- RKE2 ships Traefik with the bundled Klipper ServiceLB, so Traefik's
  external IP is a node IP. Reach the cluster from your host via that IP.
- `/etc/hosts` on your machine (example, replace with the real node IP):
  `192.168.1.10 staging.astrolumina.ro astrolumina.ro dashboard.astrolumina.ro`
- Development: `http://<node-ip>:30080` (frontend),
  `30301/30302/30303` for astrology / payment / booking.
- Staging: `http://staging.astrolumina.ro` (port 80).
- Production: `https://astrolumina.ro` (port 443, LE certificate).
- Let's Encrypt prerequisite: RKE2 installs Traefik from a HelmChart, so the
  `letsencrypt` certResolver (email `admin@astrolumina.com`) must be added to
  the Traefik `HelmChartConfig` values before applying production, e.g.
  `/var/lib/rancher/rke2/server/manifests/traefik-config.yaml` with
  `certResolvers.letsencrypt` (httpChallenge on `web`) and
  `persistence` enabled for `/certs/acme.json` equivalent storage.
- HPA needs metrics-server (not bundled with RKE2 by default).

## Deploy

```bash
kubectl apply -k development/   # dev
kubectl apply -k staging/       # staging (blue live by default)
kubectl apply -k production/    # production (blue live by default)
```
