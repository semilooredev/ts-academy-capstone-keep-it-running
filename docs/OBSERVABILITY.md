# Observability guide — how each signal flows

Capstone 2 adds three signals to Robot Shop on the Capstone 1 cluster
(the local kind cluster, context `kind-semiloore-capstone`) —
**metrics, logs, traces** — plus an availability SLO with Slack alerting.
Everything is installed by one umbrella chart: `helm/observability/`
(release `observability`, namespace `monitoring`).

```text
                          ┌──────────────────── monitoring namespace ───────────────────┐
 Robot Shop (robot-shop)  │                                                             │
                          │   kube-prometheus-stack                                     │
 web ── nginx exporter ──►│     Prometheus ──► Grafana (dashboard: robot-shop.json)     │
 (sidecar :9113)          │          ▲              ▲ datasources: Prometheus/Loki/Jaeger│
                          │          │              │                                   │
 shipping ── OTel agent ─►│   OTel Collector ── spanmetrics ──► Prometheus            │
        (OTLP gRPC :4317) │        │                                                    │
                          │        └── OTLP ──► Jaeger all-in-one ──► Jaeger UI :16686  │
 all pods ── logs ───────►│   Fluent Bit (DaemonSet) ──► Loki (:3100) ──► Grafana      │
                          │                                                             │
 web probe (30 s) ───────►│   Blackbox Exporter ──► Prometheus ──► Alertmanager ──► Slack #alerts
                          └─────────────────────────────────────────────────────────────┘
```

Pinned dependency versions (`helm/observability/Chart.yaml` + `Chart.lock`):
kube-prometheus-stack 65.5.1 · Loki 6.18.0 · Fluent Bit 0.48.4 ·
Jaeger 3.4.1 · OpenTelemetry Collector 0.108.0.

## Metrics & the Grafana dashboard

Two metric sources describe Robot Shop traffic:

1. **Web (NGINX).** The upstream web container already exposes `stub_status`
   at `/nginx_status`. The vendored Robot Shop chart adds an
   `nginx/nginx-prometheus-exporter:1.3.0` sidecar (port 9113, Service port
   name `metrics`), scraped by the `robot-shop-web` ServiceMonitor.
   Key series: `nginx_http_requests_total`.
2. **Shipping (traces → spanmetrics).** The Shipping service is instrumented
   with the OpenTelemetry Java agent (below). The Collector's **spanmetrics
   connector** turns spans into `calls_total` and
   `duration_milliseconds_bucket` series, exported on the Collector's
   `spanmetrics` port (8889) and scraped by the `otel-collector-spanmetrics`
   ServiceMonitor. Labels include `service_name`, `span_name`, `span_kind`,
   `status_code`.

The dashboard **"Robot Shop — Requests, Errors, Latency"**
(`helm/observability/dashboards/robot-shop.json`, auto-loaded by Grafana's
dashboard sidecar from a ConfigMap labelled `grafana_dashboard: "1"`) shows:

- availability SLI (30-day and 1-hour) and total NGINX requests;
- **request rate** by span: `sum by (service_name, span_name) (rate(calls_total[5m]))`
- **error ratio** by span: error-status calls ÷ all calls;
- **latency** p50/p95/p99: `histogram_quantile(...)` over
  `duration_milliseconds_bucket` (buckets are milliseconds; panels divide by
  1000 to display seconds);
- NGINX request rate and the top spans by calls.

> Live-check note: the dashboard matches the spanmetrics metric names with a
> regex (`{__name__=~"(.+_)?calls_total"}`), so a namespace prefix cannot
> silently break it. On the live cluster, open Grafana **Explore**, run
> `{__name__=~".*calls_total"}` and confirm the exact emitted name and the
> `status_code` label spelling — that 30-second check is part of the demo
> script, and it is the honest way to verify a pipeline you cannot run
> before the cluster exists.

Access Grafana:

```bash
kubectl -n monitoring port-forward svc/observability-grafana 3000:80
# http://localhost:3000  —  user: admin  —  password: capstone-admin (demo only)
```

## Logs — Fluent Bit → Loki

The Fluent Bit DaemonSet tails `/var/log/containers/*.log`, enriches each
record with Kubernetes metadata, and pushes to Loki
(`observability-loki:3100`, SingleBinary mode, filesystem storage, 10 Gi PVC).
Labels kept on every log line: `job`, `namespace`, `pod`, `container`,
`service` (from the pod's upstream `service` label).

Sample LogQL queries (Grafana → Explore → datasource **Loki**):

```logql
# Everything from Robot Shop
{namespace="robot-shop"}

# One service — the web frontend
{namespace="robot-shop", service="web"}

# Errors anywhere in the shop
{namespace="robot-shop"} |= "error"

# Payment service, JSON-parsed, only lines mentioning "failed"
{namespace="robot-shop", service="payment"} | json | __error__="" |= "failed"

# Log volume per service over 5 minutes
sum by (service) (rate({namespace="robot-shop"}[5m]))
```

**Evidence:** screenshot of the second query returning web logs.

## Traces — OpenTelemetry → Jaeger

Instrumentation is zero-code. The vendored `shipping-deployment.yaml` adds:

- an init container that downloads the pinned OpenTelemetry Java agent
  (`v2.10.0`, from the upstream OpenTelemetry GitHub release) into a shared
  `emptyDir`;
- `JAVA_TOOL_OPTIONS=-javaagent:/otel/opentelemetry-javaagent.jar` and
  `OTEL_SERVICE_NAME=shipping`, OTLP/gRPC exporter pointing at
  `observability-opentelemetry-collector.monitoring.svc.cluster.local:4317`,
  traces-only (metrics/logs exporters disabled), sampler
  `parentbased_traceidratio` at 1.0 so the demo never misses a trace.

The Collector (`alternateConfig` in `values.yaml` — used because Helm cannot
subtract default subchart config by merging) runs two pipelines:

- **traces:** OTLP in → memory limiter / resource / batch → Jaeger (OTLP) and
  → spanmetrics connector;
- **metrics:** spanmetrics → Prometheus exporter on :8889.

Jaeger is all-in-one with in-memory storage: fine for "produce one trace",
and the trace disappears if the pod restarts — capture the screenshot
promptly.

### Generate traffic and find the trace

```bash
kubectl apply -f loadgen/load-deployment.yaml -n robot-shop   # Locust, 15 clients
kubectl -n robot-shop get pods -l service=load
kubectl -n monitoring port-forward svc/observability-jaeger-query 16686:16686
```

1. Open <http://localhost:16686>, choose service **shipping**.
2. The load generator's flow (login → catalogue → cart → shipping quote →
   checkout) exercises Shipping continuously, so traces appear within a
   minute or two. Click a trace to show the span timeline.
3. Stop the load when finished: `kubectl -n robot-shop scale deployment/load --replicas=0`.

**Evidence:** screenshot of the Jaeger trace detail for `shipping`.

## Alerting

Availability probing, the SLO, burn-rate math and the Slack wiring are
documented in [`SLO.md`](SLO.md); incident response steps are in
[`RUNBOOK.md`](RUNBOOK.md).
