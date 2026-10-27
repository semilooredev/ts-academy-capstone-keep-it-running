# Blameless postmortem — template + worked example

> **How to use this file.** Part A is the blank template: copy it for the real
> instructor-led incident drill and replace every `[BRACKETED]` field with
> what actually happened. Part B is a **worked example** based on this
> stack's own alert test (Robot Shop `web` scaled to zero) so the format,
> tone and level of detail are obvious. Part B is an example, **not** a
> record of a real incident — Jeff replaces it with his real drill notes.
>
> Blameless rules: describe systems and timelines, never people at fault;
> assume everyone acted on the information they had; every action item gets
> an owner and a date.

---

## Part A — Template (copy for the real drill)

**Title:** [Incident name]
**Date:** [YYYY-MM-DD] · **Severity:** [SEV1/SEV2/SEV3] · **Status:** [Draft/Final]
**Incident lead:** [name] · **Responders:** [names]

### Summary
[Two or three sentences: what broke, who/what was affected, how it was fixed.]

### Impact
- [User-visible effect, e.g. "storefront unreachable"]
- [Duration: start–end, and how that maps to the error budget in SLO.md]

### Timeline (all times WAT)
| Time | Event |
|---|---|
| [HH:MM] | [Fault introduced / drill started] |
| [HH:MM] | [Alert fired / first symptom noticed] |
| [HH:MM] | [Diagnosis step that found the cause] |
| [HH:MM] | [Mitigation applied] |
| [HH:MM] | [Service confirmed recovered; alert resolved] |

### Root cause
[The technical mechanism, end to end. Five-whys if useful.]

### What went well
- […]

### What went poorly
- […]

### Action items
| # | Action | Owner | Due |
|---|---|---|---|
| 1 | […] | […] | […] |

---

## Part B — Worked example (replace with the real drill record)

**Title:** Robot Shop storefront unavailable — availability alert drill
**Date:** [EXAMPLE — drill date] · **Severity:** SEV2 · **Status:** Example
**Incident lead:** Semiloore Adegoke · **Responders:** Semiloore + instructor

### Summary
As a planned drill, the Robot Shop `web` deployment was scaled to zero
replicas. The synthetic availability probe began failing immediately, the
fast burn-rate alert fired to Slack #alerts, and the storefront was restored
by scaling the deployment back to one replica. The drill proved the whole
chain — probe → Prometheus rule → Alertmanager → Slack → runbook → recovery —
works end to end.

### Impact
- Storefront (`web` service) unreachable for the duration of the drill
  (example duration: ~12 minutes).
- Drill impact on the SLO: 12 minutes ≈ 24 failed probes ≈ 28% of the
  30-day error budget of 43.2 minutes / ~86 failed probes (see `SLO.md`).
  On a production service this single incident would consume over a quarter
  of the monthly budget — the drill makes the cost of downtime concrete.

### Timeline (all times WAT — example times)
| Time | Event |
|---|---|
| 14:00 | Drill starts: `kubectl -n robot-shop scale deployment/web --replicas=0` |
| 14:01 | First failed probes: `probe_success` drops to 0 |
| 14:04 | `RobotShopAvailabilityFastBurn` fires (error ratio > 1.44% over both 1 h and 5 m windows); Slack message lands in #alerts |
| 14:05 | Runbook §1 started: `kubectl get pods -n robot-shop` shows web at 0/0 replicas — cause identified as replica count, not a crash |
| 14:07 | Mitigation: `kubectl -n robot-shop scale deployment/web --replicas=1` |
| 14:09 | Web pod Ready; probe returns to 1 |
| 14:12 | Alert resolved; resolved notification lands in #alerts |

### Root cause
The deployment's desired replica count was set to zero, so the `web` Service
had no endpoints and every probe received a connection failure. No code,
image or infrastructure fault was involved — by design, the drill exercised
detection and response rather than a software bug. (In the real drill, this
section records whatever the instructor actually broke.)

### What went well
- Detection was automatic and fast: ~4 minutes from fault to a Slack page,
  with no human watching a dashboard.
- The alert text named the SLO and pointed at the runbook, so the first
  diagnostic command was already decided.
- Recovery was a single command and the resolved notification closed the loop.

### What went poorly
- The 5-minute burn window means very short outages (< ~2 minutes) resolve
  before the fast alert fires — acceptable for this SLO, but worth knowing.
- Jaeger's in-memory storage meant traces from before the drill could not be
  compared afterwards; for a longer-lived stack, persistent trace storage
  would help incident analysis.

### Action items
| # | Action | Owner | Due |
|---|---|---|---|
| 1 | Add a `RobotShopProbeDown` alert for "probe itself failing" (no data) so a broken probe cannot masquerade as a healthy service | Semiloore | [date] |
| 2 | Record the drill's screenshots (firing + resolved Slack messages) with the submission evidence | Semiloore | [date] |
| 3 | If the stack lives beyond the capstone, raise Prometheus retention to 30 days so the SLO window is fully measurable | Semiloore | [date] |
