# Capstone 2 — "Keep It Running"

**TS Academy DevOps Capstone (Beginner Edition) · Semiloore Adegoke (`semilooredev`)**
Repository: <https://github.com/semilooredev/ts-academy-capstone-keep-it-running>

Capstone 1 built and shipped an application. This repository keeps one
running: **Stan's Robot Shop** (`instana/robot-shop`) deployed onto the same
Terraform-provisioned cluster — the local **kind** cluster, context
`kind-semiloore-capstone` (see the cluster deviation note in Capstone 1's
README: the brief-literal EKS module is delivered as validated code; the
applied cluster is local because no cloud billing method was available).
The shop is wrapped in metrics, logs, traces, an availability SLO with Slack
alerting, and the runbook/postmortem paperwork of a real operations practice.

| # | Brief step | Where it lives |
|---|------------|----------------|
| 1 | Deploy Robot Shop + load generation | `helm/robot-shop/` (vendored upstream chart + `values-capstone.yaml`), `loadgen/load-deployment.yaml` |
| 2 | kube-prometheus-stack dashboard: requests, errors, latency | `helm/observability/` + `helm/observability/dashboards/robot-shop.json` |
| 3 | Fluent Bit → Loki logging + sample query | `helm/observability/values.yaml` (fluent-bit, loki), `docs/OBSERVABILITY.md` |
| 4 | OpenTelemetry trace in Jaeger | Shipping agent injection in `helm/robot-shop/templates/shipping-deployment.yaml`, Collector + Jaeger in `helm/observability/values.yaml`, walkthrough in `docs/OBSERVABILITY.md` |
| 5 | Availability SLO + error budget + Slack alert | `docs/SLO.md`, `helm/observability/templates/probe.yaml`, `helm/observability/templates/prometheus-rule.yaml`, Alertmanager config in `values.yaml` |
| 6 | Incident drill + blameless postmortem | `docs/RUNBOOK.md`, `docs/POSTMORTEM.md` (template + clearly labelled worked example) |

The click-by-click run order (including creating the Slack webhook secret) is
in [`../JEFF-ACTIONS.md`](../JEFF-ACTIONS.md); the spoken walkthrough is in
[`../DEMO-SCRIPT.md`](../DEMO-SCRIPT.md).

## Layout

```text
capstone2/
├── helm/
│   ├── robot-shop/       # vendored upstream chart (Apache-2.0, see UPSTREAM-LICENSE)
│   │   ├── values.yaml            # upstream defaults + capstone sections
│   │   ├── values-capstone.yaml   # Jeff's overrides: image 2.1.0, gp3 (EKS), exporter, OTel
│   │   └── values-kind.yaml       # kind override: Redis storage class "standard"
│   └── observability/    # umbrella chart: kube-prometheus-stack, Loki, Fluent Bit,
│       ├── Chart.yaml             # Jaeger, OTel Collector (all version-pinned)
│       ├── values.yaml
│       ├── dashboards/robot-shop.json
│       └── templates/    # dashboard ConfigMap, ServiceMonitors, Probe,
│                         # PrometheusRule burn alerts, Blackbox Exporter
├── loadgen/load-deployment.yaml   # upstream Locust load generator, image pinned 2.1.0
└── docs/
    ├── OBSERVABILITY.md  # signal flow, LogQL samples, how to find the Jaeger trace
    ├── SLO.md            # SLI, 99.9% target, error-budget math, burn rates
    ├── RUNBOOK.md        # incident response, alert by alert
    └── POSTMORTEM.md     # blank template + worked example (replace with drill notes)
```

## Local changes to the upstream Robot Shop chart

Kept deliberately small and auditable:

1. All images pinned: `robotshop/rs-*:2.1.0`, load generator `rs-load:2.1.0`
   (upstream default was `:latest`).
2. Redis StorageClass `standard` (Minikube) → `gp3` in `values-capstone.yaml`
   (the EKS setting); `values-kind.yaml` layers it back to kind's `standard`
   local-path class for the cluster actually applied.
3. `web`: added an `nginx/nginx-prometheus-exporter:1.3.0` sidecar scraping
   the already-exposed `/nginx_status`, plus a `metrics` port (9113) on the
   pod and Service.
4. `shipping`: optional OpenTelemetry Java-agent injection (init container
   downloads the pinned v2.10.0 agent; service name `shipping`; OTLP/gRPC to
   the Collector in `monitoring`).

## Deploy (summary — full steps in JEFF-ACTIONS.md)

```bash
# 0. Slack webhook secret (once) — URL is never committed
kubectl -n monitoring create namespace monitoring   # if it does not exist yet
kubectl -n monitoring create secret generic slack-webhook \
  --from-literal=webhook-url='PASTE_YOUR_SLACK_INCOMING_WEBHOOK_URL'

# 1. Observability stack
cd helm/observability
helm dependency update .
helm upgrade --install observability . --namespace monitoring --create-namespace
#    Laptop RAM tight? Add:  -f values-laptop.yaml
#    (drops node-exporter/kube-state-metrics, shrinks requests - see that file)

# 2. Robot Shop (values-kind.yaml switches Redis to kind's "standard"
#    local-path StorageClass - the gp3 class in values-capstone.yaml is the
#    EKS setting and does not exist on kind)
helm upgrade --install robot-shop ../robot-shop \
  -f ../robot-shop/values-capstone.yaml \
  -f ../robot-shop/values-kind.yaml \
  --namespace robot-shop --create-namespace

# 3. Traffic
kubectl apply -f ../../loadgen/load-deployment.yaml -n robot-shop
```

Then: Grafana dashboard for requests/errors/latency, a Loki query, one
Jaeger trace for `shipping`, and the Slack alert test — each with its
evidence screenshot listed in `../JEFF-ACTIONS.md`.

## Attribution

Robot Shop is vendored from
[instana/robot-shop](https://github.com/instana/robot-shop) (Apache License
2.0 — see `UPSTREAM-LICENSE`). The observability umbrella chart composes the
upstream Prometheus Community, Grafana, Fluent, Jaeger and OpenTelemetry
Helm charts at the pinned versions in `helm/observability/Chart.yaml`.
