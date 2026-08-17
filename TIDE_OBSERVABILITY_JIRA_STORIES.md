# TIDE Observability — Jira Story Pack (7 stories)

**Epic:** DP-118 — TIDE, CDP Kafka Streaming to Iceberg Delivery Engine
**Format:** Each story follows the house structure — Summary / Context / Acceptance criteria / Notes / Benefit statement. Copy each block into the Jira description; the first line is the story title (Jira Summary field).
**Milestone assignment:** left to backlog owner. Suggested mapping and absorbed tickets are listed per story so this pack can serve as the re-cut of the existing Milestone 2 items rather than a parallel backlog.

---

## Story 1 — Pipeline Health Dashboard: single pane of glass

**Suggested title:** `Observability - Pipeline Health Dashboard - Replicator and Snowflake Sink Single Pane of Glass`

### Summary
One Grafana dashboard that answers "is every TIDE pipeline healthy right now?" For each dataflow: Replicator status, Snowflake sink connector status, throughput in msgs/sec and MB/sec, and lag — side by side on one screen.

### Context
Today, checking pipeline health means REST calls or CLI per connector. An on-call engineer should identify a sick pipeline in under 30 seconds. Key design point: connector status alone is not health — the dashboard pairs status with throughput and lag so "running but not moving" is immediately visible.

### Acceptance criteria
- Every production pipeline appears with Replicator status, sink status, msgs/sec, MB/sec, and lag on a single screen.
- A failed or degraded component is visually unmissable (status grid, severity colors).
- Fleet-level summary tiles: pipelines live / stalled / failed, aggregate msgs/sec and MB/sec.
- Data refreshes within 60 seconds.
- Drill-through from any pipeline row to the detailed connector/task view.
- Dashboard is defined as code; no hand-built panels.

### Notes
Metrics sourced from Connect worker JMX, scraped by Alloy into cloud Grafana. Absorbs DPCS-187 (Snowflake connector throughput), DPCS-188 (Replicator throughput), DPCS-190 (connector and task status) — in-flight work on those tickets carries forward under this story.

### Benefit statement
- Operators localize a failing pipeline in seconds instead of minutes of CLI archaeology.
- Foundation layer every subsequent observability story builds on.

---

## Story 2 — Topic Coverage and Onboarding Status: all topics vs actively delivered

**Suggested title:** `Observability - Topic Coverage and Onboarding Status - Available vs Live vs Stalled`

### Summary
An automated inventory answering two questions at any moment: which topics exist on EDIA and GKOP that TIDE could deliver, and which are actually flowing to Iceberg. Every topic carries exactly one state — Available, Provisioning, Live, Live-Idle, Stalled, or Failed — plus stage tracking for in-flight self-service onboardings.

### Context
A connector reporting RUNNING is not proof data is flowing. "Live" means sink offsets are advancing while the source has traffic. "Stalled" — source active, delivery frozen — is the silent-failure state this story exists to expose; status-based monitoring never catches it. The same inventory is the source of the coverage/adoption KPI.

### Acceptance criteria
- All topics across EDIA and GKOP are inventoried automatically, with internal/system topics excluded, and each assigned one state.
- Stalled detection: source traffic present and sink offsets not advancing flags the topic within 10 minutes.
- Idle is distinguished from stalled — a topic with no source traffic and a healthy canary shows Live-Idle, never a false stale alarm.
- Coverage % (live ÷ eligible) is displayed and trends over time.
- Self-service onboarding requests are visible by stage (requested → provisioning → live), and a failed provisioning shows which stage failed.
- Inventory refreshes at least every 5 minutes; state history is retained for trend reporting.

### Notes
Requires a lightweight inventory reconciler joining cluster metadata, connector configs/status, and the self-service spec repo — design deep-dive to follow before build. Absorbs DPCS-189 (self-service provisioning status metrics).

### Benefit statement
- One source of truth for leadership's adoption question and ops' "is it actually flowing" question.
- Eliminates the silent-failure class where a pipeline looks healthy while delivering nothing.

---

## Story 3 — End-to-End Latency and Freshness: publish-to-queryable percentiles

**Suggested title:** `Observability - End to End Latency - Publish to Queryable Percentiles and Canary`

### Summary
Measure the number TIDE is judged on: time from an event published to Kafka to the row being queryable in Iceberg, as true percentiles (p50/p95/p99) per topic, refreshed every 5 minutes. A canary heartbeat per source cluster runs through the full path continuously so delivery is measured even when business traffic is quiet.

### Context
This story states the finding that closes the DPCS-192 research: Kafka and Connect JMX expose rates, averages, and maxima — not distributions. True percentiles must be computed from timestamps carried in the data itself (Kafka record timestamp compared to Iceberg load time), calculated Snowflake-side and surfaced to the dashboard. Two configuration gates must be verified before build: (1) Replicator preserves source record timestamps across the replication hop; (2) the v4 connector surfaces record metadata including the Kafka timestamp. Fallback if (2) fails: mandate a producer event timestamp field via the data contract.

### Acceptance criteria
- Every live topic shows p50/p95/p99 publish-to-queryable latency, recomputed at least every 5 minutes.
- Percentage of events under 5 minutes is visible per topic and fleet-wide — this is exactly the KR 1.3 measure ("95% of events arriving in under 5 minutes").
- A canary topic per source cluster flows through the full path (replication → sink → Iceberg) on a fixed interval.
- A late or missing canary raises an alarm.
- Latency history is retained at least 90 days for trend and SLO reporting.
- Both configuration gates above are verified and documented before the pipeline work starts.

### Notes
Latency computation runs Snowflake-side on a schedule; results exported to Grafana so alerting stays in one place. Closes DPCS-192.

### Benefit statement
- The platform's headline commitment becomes continuously measured rather than estimated.
- Directly instruments OKR KR 1.3 with no manual measurement.

---

## Story 4 — Team Dashboards: domain-scoped views with RBAC

**Suggested title:** `Observability - Team Dashboards - Domain Scoped Views Mapped to AD Groups`

### Summary
Every publishing domain (for example, EDIA FX) gets its own dashboard showing only its topics — health, throughput, freshness, DLQ — generated from a single template when the dataflow onboards, with viewing rights mapped to that team's AD group.

### Context
Teams should answer "is my topic OK" themselves, without seeing other domains' data and without a platform engineer in the loop. This depends on naming and label conventions: the self-service generator controls topic and connector names, so names must be parseable into domain/dataflow labels the dashboard template filters on. Nobody hand-builds a team dashboard, ever.

### Acceptance criteria
- Onboarding a dataflow auto-creates (or updates) its domain dashboard from the shared template — zero manual dashboard steps.
- Grafana folder permissions are bound to the owning team's AD group; team members see only their domain's folder.
- The dashboard shows, for the team's topics only: state, throughput, last-row freshness, latency percentiles, and DLQ count.
- The team can jump to logs filtered to their connectors from the dashboard without knowing which worker runs their tasks.
- Naming/label convention for topics and connectors is documented and enforced by the self-service generator.

### Notes
Verify our cloud Grafana tier supports external group sync for team RBAC before build. Absorbs DPCS-191 (team-specific dashboard links).

### Benefit statement
- Publishing teams self-serve their pipeline visibility, scoped to their own data.
- Platform team exits the support loop for routine health questions.

---

## Story 5 — Connect Cluster Capacity: headroom and scale signals

**Suggested title:** `Observability - Connect Cluster Capacity - Headroom Metric and Scale Signals`

### Summary
A capacity dashboard that tells us the Connect cluster needs to scale before consumers feel it: worker CPU, heap, task-slot utilization, rebalance rate, and — the key number — current fleet throughput against the per-task ceiling measured in performance testing. Headroom below an agreed threshold raises a ticket, and the same headroom number is queryable by the provisioning pipeline as a guardrail.

### Context
The performance-testing ceiling (DPCS-195/196 outputs) becomes the denominator for a headroom metric. That single metric serves two consumers: the ops dashboard/alert, and the deployment pipeline's capacity check (DPCS-210), which consults it before applying a new connector — so self-service onboarding can never oversubscribe the cluster. Thresholds stay open intentionally until the benchmarks land.

### Acceptance criteria
- Headroom % (current fleet demand ÷ benchmarked ceiling) is computed, displayed, and alerts below the agreed threshold once ratified.
- Per-worker CPU, heap, GC, and task distribution appear on one screen.
- Rebalance frequency and duration are tracked, with a rebalance-storm alert condition.
- Headroom is exposed via a programmatic query the provisioning pipeline can call as its capacity gate.
- Threshold values are documented as "to ratify after performance testing" until PI 3.3 benchmark results exist.

### Notes
Consumes DPCS-195/196 benchmark outputs; feeds DPCS-210 pipeline guardrails. Those tickets remain; this story provides the measurement layer between them.

### Benefit statement
- Cluster scales ahead of demand instead of after incidents.
- Capacity guardrail makes self-service onboarding safe by construction.

---

## Story 6 — Alerting and Runbooks: a page means act now

**Suggested title:** `Observability - Alerting and Runbooks - Symptom Pages, Cause Tickets, Runbook per Alert`

### Summary
A small, deliberate alert catalog (roughly 12–15 rules) defined as code. Pages for symptoms: canary delivery breached, pipeline stalled, connector in a restart loop, or a new schema version arriving with no matching Iceberg DDL. Tickets for causes: lag growing, DLQ inflow, shrinking capacity headroom. Every alert links its runbook; runbooks live in Git next to the rules. Pages route to xMatters, tickets to Slack/Jira.

### Context
With connector v4 the team owns Iceberg table DDL — automatic schema evolution is gone. A producer shipping a new field with no matching table change is the most likely production incident class TIDE will see, and by default it presents as silent DLQ growth or a stalled pipeline. It must page. Flap discipline: a single task restart stays silent; a restart loop pages.

### Acceptance criteria
- Alert catalog exists as code with severity and routing per rule.
- Page-severity rules include at minimum: canary delivery breach, pipeline stalled, connector restart loop, schema-version vs table-DDL mismatch.
- Ticket-severity rules include at minimum: sustained lag growth, DLQ inflow, capacity headroom below threshold, offset-commit failures.
- Every alert carries a runbook link; runbooks are versioned in Git alongside the rules.
- Flapping is suppressed: one restart produces no alert; N restarts in M minutes pages.
- Game-day validation: a deliberately broken non-prod pipeline produces exactly the expected page and no collateral noise.

### Notes
Confirm the Grafana-to-xMatters inbound integration path for our instance. Delivers DPCS-204 (prod monitoring, runbook, alerts).

### Benefit statement
- On-call trusts every page; alert fatigue never takes root.
- Schema-drift incidents are caught before consumers notice missing data.

---

## Story 7 — Platform SLOs and Monthly KPI Scorecard

**Suggested title:** `Observability - Platform SLOs and KPI Scorecard - Published Commitments, Auto-Generated Monthly`

### Summary
Publish the numbers TIDE commits to as a product, with measurement methods, and auto-generate a monthly scorecard: freshness SLO (at least 95% of events queryable under 5 minutes — lifted directly from KR 1.3), delivery availability measured by canary delivery, onboarding lead time, and topic coverage. Targets that depend on benchmark data are explicitly marked "to ratify after PI 3.3."

### Context
Commitments only hold if they are continuously measured. Availability is measured by canary delivery, not connector status — a delivery-based number, not an infrastructure-based one. The freshness SLO's error budget is what drives the paging thresholds in the alerting story. Discipline: measurement methods are published now; targets without baselines are marked as baseline-year, never invented.

### Acceptance criteria
- Each SLO is documented with definition, measurement method, target (or an explicit baseline-year marker), and owner, visible to consumer teams.
- The monthly scorecard renders from live data with zero manual pulls.
- Error budget for the freshness SLO is computed continuously; burn rate feeds the alerting story's paging thresholds.
- Scorecard includes at minimum: freshness attainment, delivery availability, onboarding lead time, coverage % with trend, DLQ trend.

### Notes
Scorecard is a strong candidate for Streamlit in Snowflake since the latency and coverage data already lands there. SLO definitions to be ratified with platform leadership before publication. Covers the key-measurement half of DPCS-219; replay-capability observability tracked as a follow-on.

### Benefit statement
- TIDE operates as a product with published, measured commitments.
- Leadership receives a monthly one-pager nobody had to assemble.

---

## Cross-story mapping to existing tickets

| Existing ticket | Lands in |
|---|---|
| DPCS-187 — Snowflake connector throughput | Story 1 |
| DPCS-188 — Replicator throughput | Story 1 |
| DPCS-190 — Connector and task status | Story 1 |
| DPCS-189 — Self-service provisioning status metrics | Story 2 |
| DPCS-192 — Research: latency in dashboard | Story 3 (closed by its Context) |
| DPCS-191 — Team-specific dashboard links | Story 4 |
| DPCS-195/196 — Perf testing / capacity planning | Story 5 (consumes outputs) |
| DPCS-210 — Pipeline guardrails, capacity check | Story 5 (feeds the gate) |
| DPCS-204 — Prod monitoring / runbook / alerts | Story 6 |
| DPCS-219 — Identify key measurements | Story 7 |
