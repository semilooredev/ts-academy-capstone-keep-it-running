# SLO — Robot Shop web availability

**Service:** Robot Shop storefront entry point (`web` service, namespace `robot-shop`)
**Owner:** Semiloore Adegoke · **Review window:** rolling 30 days

## SLI

The Service Level Indicator is the fraction of synthetic availability probes
that succeed:

```text
SLI = successful probes / all probes        (rolling 30 days)
```

- A probe is an HTTP GET of `http://web.robot-shop.svc.cluster.local:8080/`
  every **30 seconds**, executed by the Prometheus Operator `Probe` resource
  (`helm/observability/templates/probe.yaml`, job name
  `robot-shop-web-availability`) through Blackbox Exporter
  (`prom/blackbox-exporter:v0.25.0`, module `http_2xx`).
- Success = HTTP 2xx (redirects followed). Blackbox Exporter reports the
  result as the gauge `probe_success` (1 = success, 0 = failure), so the SLI
  is the time-average of that gauge:

```promql
avg(avg_over_time(probe_success{job="robot-shop-web-availability"}[30d]))
```

A synthetic probe is the right SLI here because it measures what a user
experiences — "can the storefront be reached at all" — independently of how
much traffic happens to arrive, and it keeps measuring even at 3 a.m. when no
real user is browsing.

## SLO

> **99.9% availability over a rolling 30-day window.**

Why 99.9% and not higher: Robot Shop is a demo storefront on a small
local learning cluster (kind: 1 control-plane + 2 workers) with single replicas. 100% is not a serious target for any
real system, and 99.99% (4.3 minutes/month) would page the operator for a
single slow node reboot. 99.9% is the standard "three nines" target for a
user-facing web service at this scale: strict enough to force good habits
(probes, alerts, runbooks), loose enough to be honest.

Why not lower: at 99% the budget would allow 7.2 hours of downtime a month —
an entire business day — which would let real problems hide inside the budget.

## Error budget — the math

```text
30 days                       = 30 × 24 × 60        = 43,200 minutes
Allowed unavailability        = 100% − 99.9%        = 0.1%
Error budget                  = 0.001 × 43,200      = 43.2 minutes per 30 days

Probes per 30 days            = 43,200 × 2 (1/30 s) = 86,400 probes
Allowed failed probes         = 0.001 × 86,400      = 86.4  ≈ 86 probes
```

Budget consumed query (percentage of the 30-day budget already spent):

```promql
(1 - avg(avg_over_time(probe_success{job="robot-shop-web-availability"}[30d]))) / 0.001 * 100
```

Policy: while budget remains, ship changes freely. If the budget is exhausted
inside the window, the next change freeze applies — reliability work
(runbook fixes, replicas, resource limits) takes priority over new features
until the window recovers. For this capstone the cluster only lives for days,
so the 30-day figures demonstrate the method; the same queries and thresholds
apply unchanged to a long-lived cluster. (Prometheus retention in
`helm/observability/values.yaml` is 7 days for the demo — raise it to ≥30
days before enforcing a real 30-day window.)

## Burn-rate alerts → Slack

Burn rate = how fast the budget is being spent relative to the sustainable
rate. A burn rate of 1 exactly exhausts the budget at the end of the window.

| Alert | Condition (error ratio) | Burn rate | Budget gone in | Severity |
|---|---|---|---|---|
| `RobotShopAvailabilityFastBurn` | > 1.44% over **both** 1 h and 5 m, for 2 m | 14.4× | ≈ 2 days | critical |
| `RobotShopAvailabilitySlowBurn` | > 0.6% over **both** 6 h and 30 m, for 15 m | 6× | ≈ 5 days | warning |

The rules live in `helm/observability/templates/prometheus-rule.yaml` and use
the standard multi-window pattern (a long window confirms the trend, a short
window confirms it is still happening), so a single failed probe never pages
anyone: one failed probe is 3.3% of a 5-minute window but only 0.83% of a
1-hour window, below the fast-burn threshold.

Both alerts are routed by Alertmanager to Slack channel **#alerts**:

- Routing/receiver: `kube-prometheus-stack.alertmanager.config` in
  `helm/observability/values.yaml`.
- The webhook URL is **never committed**. Jeff creates it as a Kubernetes
  Secret and Alertmanager reads it from the mounted file:

```bash
kubectl -n monitoring create secret generic slack-webhook \
  --from-literal=webhook-url='PASTE_YOUR_SLACK_INCOMING_WEBHOOK_URL'
# referenced in values.yaml as:
#   api_url_file: /etc/alertmanager/secrets/slack-webhook/webhook-url
```

**Evidence:** screenshot of this document's numbers, the Prometheus rules page
showing both alerts, and the firing + resolved messages in #alerts during the
alert test in `JEFF-ACTIONS.md`.
