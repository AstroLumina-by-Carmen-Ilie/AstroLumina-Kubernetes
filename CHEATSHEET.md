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
