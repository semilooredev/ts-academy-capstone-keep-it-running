# Runbook — Robot Shop on Kubernetes

Audience: the on-call operator (for this capstone: Jeff). All commands assume
`kubectl` is pointed at the Capstone 1 kind cluster (context `kind-semiloore-capstone`, `KUBECONFIG` set from the local-cluster module's output). Namespaces:
`robot-shop` (application), `monitoring` (observability), `voting-app`
(Capstone 1 — not covered here).

Golden commands, in the order to run them for any incident:

```bash
kubectl get pods -n robot-shop
kubectl get pods -n monitoring
kubectl get events -n robot-shop --sort-by=.lastTimestamp | tail -20
```

---

## 1. Alert: RobotShopAvailabilityFastBurn / SlowBurn (Slack #alerts)

Meaning: the availability probe of `web` is failing — see `docs/SLO.md` for
the budget math. Fast burn = act now; slow burn = act today.

1. Check the web tier:
   ```bash
   kubectl -n robot-shop get pods -l service=web
   kubectl -n robot-shop describe pod -l service=web | tail -30
   kubectl -n robot-shop logs -l service=web -c web --tail=100
   ```
2. Check the entry point itself:
   ```bash
   kubectl -n robot-shop get svc web          # load balancer hostname present?
   kubectl -n robot-shop get endpoints web    # backends listed?
   ```
3. Check the probe path is alive (distinguish "app down" from "probe broken"):
   ```bash
   kubectl -n monitoring logs -l app.kubernetes.io/name=blackbox-exporter --tail=50
   # In Grafana Explore (Prometheus): probe_success{job="robot-shop-web-availability"}
   ```
4. If a node is the problem:
   ```bash
   kubectl get nodes
   kubectl describe node <node> | grep -A5 Conditions
   ```
5. Mitigate: restart a wedged pod with
   `kubectl -n robot-shop rollout restart deployment/web`, or roll back the
   last Robot Shop release with `helm rollback robot-shop -n robot-shop`.
6. Confirm recovery: probe returns to 1, the Alertmanager alert resolves
   (a resolved message lands in #alerts), then write up the incident using
   `docs/POSTMORTEM.md`.

## 2. Pod stuck in ImagePullBackOff

```bash
kubectl -n robot-shop describe pod <pod> | grep -A3 Events
```

- Robot Shop images are pinned to `robotshop/rs-*:2.1.0` in
  `helm/robot-shop/values-capstone.yaml` — a typo'd tag is the usual cause.
- If the events mention the Capstone 1 GHCR images instead, the package is
  probably still private: make it public (package → Package settings →
  Change visibility) or add an image-pull Secret to `image.pullSecrets`.

## 3. Pod stuck in Pending — PVC not binding

```bash
kubectl -n robot-shop get pvc
kubectl get storageclass
```

- On the **kind cluster** the default StorageClass is `standard`
  (local-path). Robot Shop Redis must use it: the install layers
  `values-kind.yaml` after `values-capstone.yaml` for exactly this. A PVC
  asking for `gp3` here will never bind — reinstall with both files.
- Check the local-path provisioner is running:
  `kubectl -n local-path-storage get pods`.
- (On the EKS module the equivalent failure is a missing EBS CSI driver —
  Terraform installs it as an add-on — or a missing default class; see
  `capstone1/terraform/gp3-storageclass.yaml`.)

## 4. Grafana dashboard is empty

1. Targets up? Grafana → Explore → Prometheus, or
   `kubectl -n monitoring port-forward svc/observability-kube-prometheus 9090:9090`
   → Status → Targets: look for `robot-shop-web` and
   `otel-collector-spanmetrics`.
2. Web metrics: `kubectl -n robot-shop logs -l service=web -c nginx-exporter`
   — the sidecar must reach `http://127.0.0.1:8080/nginx_status`.
3. Spanmetrics names: in Explore run `{__name__=~".*calls_total"}`. The
   dashboard already matches names by regex; if the series exists under a
   different label set, check the Collector (next section).

## 5. No traces in Jaeger

1. Load is running: `kubectl -n robot-shop get deployment load` (replicas ≥ 1)
   and the shop is reachable — no traffic, no traces.
2. Shipping agent attached:
   `kubectl -n robot-shop logs -l service=shipping -c shipping | grep -i opentelemetry`
   and the init container finished:
   `kubectl -n robot-shop get pod -l service=shipping` (init 1/1 complete).
3. Collector healthy and receiving:
   ```bash
   kubectl -n monitoring logs -l app.kubernetes.io/name=opentelemetry-collector --tail=100
   kubectl -n monitoring get pods -l app.kubernetes.io/name=jaeger
   ```
4. Jaeger storage is in-memory: if the Jaeger pod restarted, old traces are
   gone by design — generate fresh traffic and search again.

## 6. No logs in Loki

```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=fluent-bit
kubectl -n monitoring logs -l app.kubernetes.io/name=fluent-bit --tail=100
kubectl -n monitoring get pods -l app.kubernetes.io/name=loki
```

- Fluent Bit errors naming `observability-loki` → Loki Service is down or the
  release name changed (the output host in `values.yaml` assumes release name
  `observability`).
- Logs exist but the query is empty → check the label names in Explore's
  label browser; this stack emits `namespace`, `pod`, `container`, `service`.

## 7. Slack alerts not arriving

```bash
kubectl -n monitoring get secret slack-webhook        # must exist
kubectl -n monitoring logs -l app.kubernetes.io/name=alertmanager --tail=100
```

- Secret missing → Alertmanager cannot reconcile its config; create it
  (see `docs/SLO.md`) and wait a minute.
- Webhook wrong/revoked → regenerate the Slack incoming webhook for #alerts
  and recreate the Secret; never commit the URL to git.
- Alert is firing in Prometheus (Alerts page) but silent in Slack →
  Alertmanager → check the receiver test in the Alertmanager UI via
  `kubectl -n monitoring port-forward svc/observability-kube-prometh-alertmanager 9093:9093`.

## 8. Cost control — shutting the stack down

The cluster bills while it exists. To pause safely:

```bash
helm uninstall robot-shop -n robot-shop
helm uninstall observability -n monitoring
helm uninstall voting-app -n voting-app
cd ../capstone1/terraform/local-cluster && terraform destroy
# $0 cluster - nothing bills - but destroying also proves the lifecycle
# and frees the laptop's RAM.
```
