# AstroLumina Kubernetes — Cheatsheet

## All application logs at once (with pod prefix)

`kubectl logs` accepts a label selector, so one invocation tails every
matching pod in parallel. `--prefix` prepends each line with `pod/<name>`.
`--max-log-requests` must cover the pod count (24+ pods live in
staging/production when both colors run).

### Development (single color)

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina -n astrolumina-dev --prefix --tail=50 --max-log-requests=30"
```

### Staging / production — both colors

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina -n astrolumina-staging --prefix --tail=50 --max-log-requests=30"
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina -n astrolumina-prod --prefix --tail=50 --max-log-requests=30"
```

### Staging / production — one color only

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina,variant=blue -n astrolumina-staging --prefix --tail=50 --max-log-requests=30"
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina,variant=green -n astrolumina-prod --prefix --tail=50 --max-log-requests=30"
```

### Follow mode (live tail)

Append `-f` to any of the above:

```bash
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina -n astrolumina-dev --prefix --tail=50 --max-log-requests=30 -f"
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina -n astrolumina-staging --prefix --tail=50 --max-log-requests=30 -f"
ssh "$K8S_CP_CONN" -- "kubectl logs -l app.kubernetes.io/part-of=astrolumina -n astrolumina-prod --prefix --tail=50 --max-log-requests=30 -f"
```

NOTE: `kubectl logs` takes exactly one resource or selector per
invocation — the label selector is the idiomatic way to get "all services
at once". The selector works because every Deployment carries the
`app.kubernetes.io/part-of=astrolumina` label (plus `variant=blue|green`
in staging/production).

## Read a Secret's real value (base64 decode)

Secret data is stored base64-encoded. List key names only (safe to share),
then decode one key:

```bash
ssh "$K8S_CP_CONN" -- "kubectl get secret env-frontend-secrets -n astrolumina-dev -o json | python3 -c 'import json,sys; print(sorted(json.load(sys.stdin)[\"data\"]))'"
ssh "$K8S_CP_CONN" -- "kubectl get secret env-frontend-secrets -n astrolumina-dev -o jsonpath='{.data.STRIPE_PK}' | base64 -d; echo"
```

The `; echo` adds the trailing newline `base64 -d` omits. Never paste
decoded secrets into chat, tickets, or the repo — read, use, forget.

## Exec into a pod (run a command inside)

By pod name (get it from `kubectl get pods -n <ns>` first):

```bash
ssh "$K8S_CP_CONN" -- "kubectl exec -n astrolumina-dev <pod-name> -- cat /usr/share/nginx/html/env.js"
```

Per Deployment — no pod name needed, runs in the FIRST pod (handy for a
quick file check on the frontend):

```bash
ssh "$K8S_CP_CONN" -- "kubectl exec -n astrolumina-dev deploy/frontend -- cat /usr/share/nginx/html/env.js"
ssh "$K8S_CP_CONN" -- "kubectl exec -n astrolumina-staging deploy/frontend-blue -- env | sort"
```

Interactive shell (when the image has one — nginx-based images usually do
not ship `bash`, try `sh`):

```bash
ssh "$K8S_CP_CONN" -- "kubectl exec -n astrolumina-dev deploy/frontend -it -- sh"
```

NOTE: `-it` needs a real terminal — run that last one from an
interactive ssh session on the control plane, not through a quoted
one-liner.
