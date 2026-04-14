# SLA / Monitoring System Redesign

**Status**: Design proposal — awaiting implementation approval

**Author**: Zane + Claude Opus 4.6 (1M context)

**Created**: 2026-04-14

**Scope**: Refactor / partial rewrite of `monitoring_*` backend tables, `/api/v1/monitoring/*` endpoints, `/admin/vio-monitoring/*` frontend, and probe cron scripts on Amy/Eva/Vio EC2s.

**Decision**: **Option B' — Hybrid Refactor (precision-cut)** — see §5.

---

## 0. TL;DR

The current "multi-layer SLA system" is **over-engineered and partially broken**:

- **98.5%** of probe volume is L1–L4 infrastructure noise already covered by New Relic APM + CloudWatch.
    
- **100% of `amy` e2e_synthetic probes have been `timeout` for 7+ days** — the exact thing the system is supposed to catch — and **no alert fired** because the analysis engine has been dead for 40 days (`monitoring_sla_snapshots` last written 2026-03-04).
    
- Storage is currently manageable (**2.86 GB on a 135 GB shared RDS**), but grows **~21 GB/year** uncleaned. A `/cleanup` endpoint exists but has **never been scheduled**: 1.5M rows (25% of the table) are past the 30-day retention window.
    
- 2 of 3 large indexes (1 GB combined) are nearly unused, paying write-cost for zero read-benefit.
    

**The real requirement**, based on a direct Q&A with the owner:

> "We don't have client SLA contracts. The UI is only checked occasionally. What we really need is **reliable alerting**."

This reframes the project from "SLA system" to "**alerting system + a minimal debug tail**" — which changes the design materially.

**Recommended path (Option B')**: delete L1-L4 probes, delete SLA report module, delete frontend, keep only L5/L6 synthetic chatbot probes + a dedicated pg_cron alerting path to Lark, add Sentry for backend exceptions, activate New Relic Alerts for APM-side coverage.

Net effect:

- **−1,800 lines of code**
    
- **−1.5 GB storage immediately, −21 GB/year forever**
    
- Alerting latency **hours-to-never → <15 minutes**
    
- Zero new recurring cost (uses already-paid tools)
    

---

## 1. Context — How We Got Here

### 1.1 The system's original intent

The current `monitoring_*` schema was built to answer four different questions with one system:

```Plain
| # | Question | Who asks it | Success criterion |
|---|----------|-------------|-------------------|
| A | "Who broke? Which layer?" | On-call engineer | Diagnose in <5 min |
| B | "Can the bot still reply to a user message?" | Platform team | Detect outage <1 min |
| C | "Did we meet 99.9% this month?" | BD / Clients | Monthly report |
| D | "Wake someone up when X happens" | Ops | Push to Lark/Slack within seconds |
```

Conflating A–D into one "multi-layer SLA system" is the root cause of the mess. Each has a **different optimal tool**, and the self-built one-size-fits-all attempt ends up weak at all four.

### 1.2 Evidence of the mess (measured 2026-04-14)

```Plain
Total PostgreSQL DB:              135 GB (shared: sourcing, ads, vio, amy, monitoring)
monitoring_probe_results:         2.86 GB  / 5,901,535 rows  / 138,700 rows/day
monitoring_alerts:                112 kB   / 46 rows  / last write 2026-03-04 (~40 days dead)
monitoring_sla_snapshots:         96 kB    / 32 rows  / last write 2026-03-04 (~40 days dead)
monitoring_probe_configs:         24 kB    / 0 rows   / empty since creation

Probes past 30-day retention:     1,495,455 (25% of table) — never cleaned
Index bloat (indexes/table ratio): 60%
Unused indexes (<1% of scans):    idx_mon_probe_agent_layer_time (621 MB, 102 scans)
                                  idx_mon_probe_layer_time (422 MB, 117 scans)

24h volume distribution:
  L1 platform_connectivity:       37.9%  (60,478 rows — ~all "healthy", constant noise)
  L2 gateway_ingress:             21.7%  (34,588 rows)
  L3 route_resolution:            19.0%  (30,281 rows)
  L4 agent_container:             19.9%  (31,740 rows)
  L5 message_processing:           0.7%  (1,152 rows — ALL "timeout" since 2026-04-07 02:15)
  L6 response_delivery:            0.7%  (1,152 rows — ALL "timeout" since 2026-04-07 02:15)
```

Critical finding: **the 1.4% of probe volume that actually matters (L5/L6, chatbot reply health) has been 100% broken for a week and no one noticed.** The 98.5% noise floor obscured it.

### 1.3 Session timeline of discovery

1. User asked: "can we probe PG to check why data isn't logging anymore?"
    
2. SQL revealed DB **is** logging — freshest row 22 seconds ago. But the SLA page was showing no data.
    
3. Deeper query revealed the gap: `amy` e2e_synthetic had a **44-hour blackout** 2026-04-08 02:00 → 2026-04-09 21:00 UTC.
    
4. Status breakdown revealed the catastrophe: **every probe since 2026-04-07 02:15 UTC is `timeout`**, including during the "recovered" window (5,998 probes, 0 healthy for `amy` alone).
    
5. Root cause likely PR #1780 (shadow forwarding OpenClaw → Nanobot, merged 2026-04-06 21:39 UTC, ~4.5h before cutoff) and/or PR #1774 (Lark token consolidation).
    
6. Evidence-based design decision: the monitoring system's primary failure is **no alerting** — writing rows isn't the problem, acting on them is.
    

---

## 2. Current Architecture

```Plain
┌─────────────────────────────────────────────────────────────────────┐
│ INGEST (cron-based, on each EC2)                                    │
│                                                                     │
│  Amy EC2:                                                           │
│    * * * * *  layer-health-probe.sh    → PG monitoring_probe_results│
│    */5 * * *  e2e-synthetic-probe.py   → PG monitoring_probe_results│
│                                                                     │
│  Vio EC2:                                                           │
│    * * * * *  layer-health-probe.sh    → PG monitoring_probe_results│
│    */5 * * *  e2e-synthetic-probe.py   → PG monitoring_probe_results│
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STORAGE (vizzylabs RDS, public schema)                              │
│  monitoring_probe_results   (5.9M rows, 2.86 GB, growing 60 MB/day) │
│  monitoring_alerts          (46 rows, dedup_key-based)              │
│  monitoring_sla_snapshots   (32 rows, precomputed SLA)              │
│  monitoring_probe_configs   (EMPTY — intended but never used)       │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ANALYSIS (NOT RUNNING — dead 40 days)                               │
│  POST /api/v1/monitoring/analyze                                    │
│    - Compute SLA snapshots                                          │
│    - Detect breaches → insert monitoring_alerts                     │
│    - Post to Lark webhook                                           │
│  ← No scheduler calls this                                          │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ READ PATH                                                           │
│  FastAPI /api/v1/monitoring/*  (10 endpoints, 1,605 lines)          │
│  Frontend /admin/vio-monitoring/*  (3 pages, 438 lines)             │
│    - /              overview card grid                              │
│    - /sla           table with p50/p95/p99                          │
│    - /alerts        alert list                                      │
│    - /[agent]       drill-down                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.1 What works

- `ec2_source` tagging (`amy-gateway` vs `vio-v2`) — enabled this session's forensic analysis
    
- `details` JSONB field storing `probe_id` for cross-reference
    
- `idx_mon_probe_status_time` (18,658 scans) — genuinely used
    
- Layer enum (L1–L6) is conceptually clean
    

### 2.2 What doesn't work

- **Analysis engine never scheduled** — `/analyze` is a manual button, no one presses it
    
- **Cleanup never scheduled** — `/cleanup` is a manual button, no one presses it
    
- **98.5% of probes are signal-less** (L1–L4 noise)
    
- **Dedup logic too aggressive** — 30-min cooldown + dedup_key silently suppresses real alerts
    
- **Alert channel is only Lark** — no redundancy if Lark outage
    
- **`monitoring_probe_configs` is empty** — the planned "dynamic probe configuration" feature was never built
    
- **Frontend exists but is low-value** — per owner, checked only occasionally
    

---

## 3. Available Resources (Inventory)

### 3.1 Already paid / already running

```Plain
| Tool | Status | What it can do | Currently utilized? |
|------|--------|----------------|--------------------|
| **New Relic** | FastAPI + Django production (`NEW_RELIC_LICENSE_KEY` in `.env.gcr`) | APM, error tracking, slow-query, **Synthetics**, Alerts, Dashboards, NRQL | Partial — APM yes, Alerts NO |
| **Hatchet** | Autopilot workflows | Task queue + retry + DLQ + observability | Not for monitoring |
| **Insforge** | LLM context recording | Conversation context storage | Not for monitoring |
| **AWS CloudWatch** | Default for RDS/EC2 | Infrastructure metrics, logs, alarms | Minimal |
| **Prefect** | Scheduling many flows | Retry, observability, scheduling | Not for monitoring cleanup |
```

### 3.2 Low-cost additions that fill real gaps

```Plain
| Tool | Cost | Fills gap |
|------|------|-----------|
| **Sentry** | Free tier 5K events/month (backend); mobile already using | Backend exception alerts (100% missing today) |
| **UptimeRobot / Better Uptime** | Free 50 monitors / $18 per month unlimited | External DNS/TLS/network probe (blindspot today) |
| **PagerDuty Free** | 5 users free | Real on-call rotation + escalation (we only push Lark today — misses when no one awake) |
| **Grafana Cloud Free** | 10K metrics, 50 GB logs | Secondary dashboard, alternative viz |
| **Checkly** | Free 10K API + 1K browser checks | Playwright-style synthetic monitoring |
```

### 3.3 Not applicable (evaluated honestly)

- **OpenTelemetry** — Technically the right long-term answer, but the owner confirmed team will not add DevOps/SRE in next 12 months. OTel Collector is a real operational burden. **Excluded.**
    

---

## 4. The Four Options Considered

### 4.1 Option A — Keep & Fix (minimal change)

Keep the self-built system. Fix cleanup, fix analyze cron, add probe-of-probes.

```Plain
| Dimension | Rating |
|-----------|--------|
| Effort | 40 min – 2 h |
| Code delta | +200 lines |
| Code retained | 2,042 lines |
| Yearly storage cost | Unchanged (+21 GB/year trajectory) |
| Control | ✅ Highest (all self-owned) |
| Professionalism | ❌ Low (custom implementation of a solved problem) |
| Vendor lock-in | ✅ None |
| Team cognitive load | ⚠️ Medium (must internalize 6-layer model) |
```

**Pros**: Smallest change, preserves UI, zero re-learning.

**Cons**: Continues maintenance of a monitoring stack that competes with professional tools. Degradation likely — analyze will stop again in 2 months.

### 4.2 Option B — Hybrid Refactor (delete 98.5%, keep the 1.4%)

- **Delete** L1–L4 infrastructure probes (NR APM + CloudWatch natively cover them)
    
- **Keep** L5/L6 chatbot synthetic probes (genuinely hard to replicate with off-the-shelf tools)
    
- **Simplify** frontend to 1 page focused on synthetic probe status
    
- **Route alerts** via NR (APM) + pg_cron-based direct Lark webhook (synthetic)
    

```Plain
| Dimension | Rating |
|-----------|--------|
| Effort | 1 day |
| Code delta | −1,600 / +200 = **−1,400 lines** |
| Yearly storage cost | ~500 MB, near-zero growth |
| Control | ✅ High (critical path self-owned) |
| Professionalism | ✅ High (NR does what NR does best) |
| Vendor lock-in | ⚠️ Medium (NR already there) |
| Team cognitive load | ✅ Low |
```

**Pros**: Eliminates noise while preserving genuinely unique capability. NR earns its fee.

**Cons**: Partial frontend rework. Team must learn basic NRQL.

### 4.3 Option C — Nuclear (all SaaS, full delete)

Delete entire `monitoring_*` table family, entire `/api/v1/monitoring/*`, entire `/admin/vio-monitoring/*` frontend.

```Plain
| Function | Replacement |
|----------|-------------|
| L1 external connectivity | NR Synthetics (already paid) or UptimeRobot free |
| L2 HTTP Gateway | NR APM (automatic) |
| L3 routing | NR APM custom attribute |
| L4 container health | CloudWatch Container Insights |
| L5 message processing | NR Custom Events from Amy/Vio agent code |
| L6 reply delivery | NR Custom Events + Checkly for true E2E |
| Exceptions | Sentry (backend + frontend) |
| Alerts | NR Alerts → PagerDuty free tier |
| SLA reports | NRQL monthly exports |
```

```Plain
| Dimension | Rating |
|-----------|--------|
| Effort | 4–8 h |
| Code delta | **−2,042 lines, +100 lines** |
| Yearly storage cost | ~0 (all external) |
| Control | ⚠️ Low (NR goes down, you go dark) |
| Professionalism | ✅ Highest |
| Vendor lock-in | ❌ High |
| Team cognitive load | ✅ Lowest |
```

**Pros**: 2000+ lines to zero, highest professionalism, near-zero long-term operating cost.

**Cons**: If NR outage, alerting path compromised. Loss of `/admin/vio-monitoring/` UI.

### 4.4 Option D — OpenTelemetry standardization

Use OpenTelemetry SDK in Amy/Vio/Eva → OTel Collector → multiple backends (NR, Sentry, Grafana, Prometheus).

```Plain
| Dimension | Rating |
|-----------|--------|
| Effort | 2–3 days |
| Code delta | −1,800 / +400 = −1,400 lines |
| Yearly storage cost | NR already paid + $5/mo Collector EC2 |
| Control | ✅ Highest (own the data, swap backends) |
| Professionalism | ✅ Highest (industry standard) |
| Vendor lock-in | ✅ None |
| Team cognitive load | ❌ High (OTel learning curve is steep) |
```

**Pros**: Technically cleanest. Data ownership. Portable across backends.

**Cons**: Steep learning curve. Collector is another operational component. Excluded given team headcount constraint (no SRE planned in next 12 months).

### 4.5 Decision matrix

```Plain
| Dimension | A Keep+Fix | B Hybrid | C Nuclear | D OTel |
|-----------|------------|----------|-----------|--------|
| Immediate bleed-stop | ✅ | ⚠️ | ⚠️ | ❌ |
| Long-term ops cost | ❌ | ✅ | ✅ | ⚠️ |
| Industry-standard | ❌ | ✅ | ✅ | ✅ |
| Change risk | ✅ | ⚠️ | ⚠️ | ❌ |
| Team learning load | ✅ | ⚠️ | ⚠️ | ❌ |
| Vendor lock-in | ✅ | ⚠️ | ❌ | ✅ |
| Net code reduction | ±0 | −1,400 | −2,000 | −1,400 |
| Resilience when NR down | ✅ | ✅ | ❌ | ✅ |
```

---

## 5. Owner's Requirements (clarifying Q&A)

Three questions posed to refine the target:

```Plain
| Q | Answer | Design impact |
|---|--------|---------------|
| 1. Any client contracts requiring SLA reports? | **No** | Delete SLA report module entirely (`monitoring_sla_snapshots`, `/sla` endpoint, `/sla` frontend page) |
| 2. Is `/admin/vio-monitoring/` used daily? | "Only occasionally" — core need is **alerting** | Delete frontend entirely. Fall back to `psql` / NR dashboard. A half-maintained UI is worse than none. |
| 3. Will we hire a DevOps/SRE in next 12 months? | **No** | Exclude Option D (OTel). Team cannot absorb Collector operational load. |
```

### 5.1 Refined problem statement

The owner's **single true requirement** is:

> **"When the bot breaks, wake me up in Lark within 5 minutes."**

Everything else (dashboards, reports, 6-layer hierarchy) is either nice-to-have or outright waste.

This reframing collapses the decision:

- Option A — doesn't fix alerting
    
- Option C — depends on NR alert path (NR outage → blind)
    
- Option D — excluded
    
- **Option B, further pruned** — is the answer
    

---

## 6. Recommended Plan — Option B' (precision-cut)

### 6.1 What to delete

```Plain
| Object | Lines / size | Why |
|--------|--------------|-----|
| `/admin/vio-monitoring/` frontend (three pages + layout) | 438 lines | Owner only checks occasionally; half-maintained UI is liability |
| `monitoring_sla_snapshots` table + `/sla` endpoint + `sla.py` | 141 lines + table | No client SLA contracts |
| `monitoring_probe_configs` table | Empty | Dead code since creation |
| `/analyze` endpoint + in-file analysis engine (`analysis.py` minus `/cleanup`) | ~300 lines | Hasn't run in 40 days; new pg_cron replaces |
| `layer-health-probe.sh` on Amy & Vio (L1–L4 probes) | 2 scripts | NR APM + CloudWatch natively cover these |
| Index `idx_mon_probe_agent_layer_time` (621 MB) | | 102 scans, negligible usage |
| Index `idx_mon_probe_layer_time` (422 MB) | | 117 scans, negligible usage |
```

**Total removed: ~1,800 lines of code, ~1.5 GB of data, ~1 GB of indexes, 2 cron scripts, 2 tables.**

### 6.2 What to keep

```Plain
| Object | Reason |
|--------|--------|
| `e2e-synthetic-probe.py` (Amy + Vio, already deployed) | Real Slack/Lark round-trip — NR Synthetics cannot easily replicate |
| `monitoring_probe_results` → rename to **`agent_synthetic_health`** | Core observation table; reduce to 7-day retention, ~20 MB |
| Fields: `agent_name`, `ec2_source`, `status`, `details`, `probed_at` | Validated useful this session |
| Index `idx_mon_probe_status_time` | 18,658 scans — genuinely hot path |
| `POST /cleanup` endpoint (renamed) | Re-wire to scheduled Prefect flow |
```

### 6.3 What to add

1. **pg_cron scan + Lark webhook** (~15 lines SQL + ~50 lines Python)
    

- Three trigger rules:
    
- `synthetic probe: 3 consecutive timeouts for same (agent, platform)` → critical
    
- `no new probe row for >5 minutes (writer died)` → critical
    
- `synthetic probe: timeout rate >50% over last 1 hour` → warning
    
- Direct POST to Lark webhook `LARK_OPS_WEBHOOK_URL` — **does not depend on NR**
    
- Deliberately simple, no UI, intentionally a thin wire
    

1. **Sentry integration** (backend) — free tier
    

- FastAPI: `sentry-sdk[fastapi]`
    
- Django: `sentry-sdk[django]`
    
- Backend exceptions auto-reported, UI in Sentry web app
    
- ~10 min work
    

1. **New Relic Alert policies** (zero code, NR control plane)
    

- APM error rate > 1% (5-min rolling) → Lark webhook
    
- p99 latency > 10s (any transaction) → Lark webhook
    
- Already paid — not configuring these = burning money
    

1. **Scheduled cleanup** (Prefect daily flow)
    

- `POST /api/v1/monitoring/cleanup?retention_days=7`
    
- Time: 03:00 UTC (off-peak, away from ads batch)
    

### 6.4 Target architecture

```Plain
┌────────────────────────────────────────────────────────────────────┐
│ INGEST  (one path only, each EC2 via cron)                         │
│   Amy EC2  → e2e-synthetic-probe.py  (every 5 min)                 │
│   Vio EC2  → e2e-synthetic-probe.py  (every 5 min)                 │
│                            │                                       │
│                            ▼                                       │
│   PostgreSQL: agent_synthetic_health  (7-day retention, ~20 MB)    │
└────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────────────┐
│ ALERT  (pg_cron every 15 min, independent path)                    │
│   SQL rules → Python webhook handler → Lark OPS group              │
│   Rules:                                                           │
│     • 3 consecutive timeouts                                       │
│     • 5 min without new rows (writer dead)                         │
│     • 1 h timeout rate > 50%                                       │
└────────────────────────────────────────────────────────────────────┘

┌─── Auxiliary paid-already channels (zero custom code) ────────────┐
│   NR APM Alerts   → HTTP error / p99 latency  → Lark               │
│   Sentry          → backend exceptions        → Lark / Slack       │
│   CloudWatch      → EC2 / RDS infrastructure  → existing           │
└────────────────────────────────────────────────────────────────────┘
```

Total code footprint after refactor:

- ~200 lines self-built (probe script + pg_cron rule + webhook handler)
    
- 0 new frontend code (deleted)
    
- ~3 NR console clicks + 1 Sentry DSN install
    

### 6.5 Key architectural decisions

#### Decision 1: Alerting path **bypasses** New Relic

Amy/Vio EC2 → pg_cron → Lark webhook directly. Does NOT go through NR.

- Reason: If NR has an outage, our self-built alerts still fire.
    
- NR becomes secondary (APM-side alerts). Dual path, near-zero extra cost.
    

#### Decision 2: Frontend fully deleted, not "simplified to 1 page"

- Owner confirmed "only occasionally" used.
    
- A partially-maintained UI gives false confidence. Better to have no UI and force users to use professional tools (NR dashboard, psql).
    

#### Decision 3: No OpenTelemetry

- Team will not hire DevOps/SRE in next 12 months.
    
- OTel Collector is real operational weight that we cannot absorb.
    

#### Decision 4: Keep one table and ~200 lines of self-built

- NR Synthetics cannot cleanly do "send a Slack message as user, wait for bot to reply". Checkly can but costs $40+/mo.
    
- 200 self-built lines is cheaper than $480/year Checkly, and gives us control over probe semantics.
    

### 6.6 What the owner must accept

- Losing `/admin/vio-monitoring/` UI (acceptable per Q&A).
    
- NR and Sentry become critical dependencies (but only on the auxiliary alert channel; primary path is self-owned).
    
- 30-day SLA reports will require ad-hoc NRQL if ever needed (acceptable: no client contract).
    

---

## 7. Implementation Plan — 4 PRs

Execute in order. Each PR is independently revertable. **PR #4 is destructive and must wait until PR #2 has been stable for ≥1 week in production.**

```Plain
| # | Title | Risk | Effort | Rollback |
|---|-------|------|--------|----------|
| 1 | `fix(monitoring): schedule cleanup + drop unused indexes` | 🟢 Low | 1 h | Re-create indexes from Alembic history |
| 2 | `feat(monitoring): pg_cron-based Lark alerting` | 🟢 Low | 1.5 h | Disable pg_cron job |
| 3 | `feat(backend): Sentry DSN + New Relic alert policies` | 🟢 Low | 1 h | Remove Sentry init; disable NR policies |
| 4 | `refactor(monitoring): delete L1–L4 probes, frontend, SLA reports` | 🟡 Medium (destructive) | 2 h | Revert PR; restore deleted tables from backup if needed |
```

Total: ~5.5 hours of focused work, split into 4 independently-mergeable PRs.

### 7.1 PR #1 — Bleed-stop

Goals:

- Stop unbounded storage growth
    
- Reclaim 1 GB of unused index space
    
- Reduce write-amplification (unused indexes are still updated on every INSERT)
    

Changes:

- New Prefect flow: `flows/monitoring_cleanup/flow.py`
    
- Daily at 03:00 UTC
    
- Calls `POST /api/v1/monitoring/cleanup?retention_days=7`
    
- Alerts on failure via Lark webhook
    
- Alembic migration: drop `idx_mon_probe_agent_layer_time` and `idx_mon_probe_layer_time`
    
- Note: both indexes have <0.6% of `idx_mon_probe_status_time` scan count
    
- Downside: rare queries filtering on `(agent, layer, time)` or `(layer, time)` become seqscans. Acceptable given scan count.
    

Verification:

- Before: `SELECT COUNT(*) FROM monitoring_probe_results WHERE probed_at < NOW() - INTERVAL '7 days';` returns ~4.8M
    
- After first cleanup run: same query returns <100
    
- DB size: pre-2.86 GB, post-~700 MB
    

### 7.2 PR #2 — New alert path (parallel, non-destructive)

Goals:

- Establish a reliable, low-latency alert channel that **does not depend on NR**
    

Changes:

- New file: `backend-fastapi/app/services/synthetic_health_alerter.py`
    
- 3 SQL rules (encoded)
    
- Webhook poster with dedup (in-memory 30 min cooldown)
    
- Install `pg_cron` extension in RDS (one-time, by DBA):
    
- `SELECT cron.schedule('synthetic-health-scan', '*/15 * * * *', 'SELECT ... http_post(...)')` — Option 1 (pure DB)
    
- OR: Prefect flow `flows/synthetic_health_alert/flow.py` every 15 min — Option 2 (no RDS change)
    
- **Prefer Option 2** (Prefect) — no RDS extension needed, reuses existing infra
    

Verification:

- Manually insert a fake "timeout" probe to trigger rule 1 → Lark alert fires within 15 min
    
- Kill the probe cron on Amy EC2 for 10 min → rule 2 fires
    
- Leave both running for 1 week, observe:
    
- No false positives
    
- Known real incident (any new timeout) triggers alert
    

### 7.3 PR #3 — Paid tools activation

Goals:

- Turn on tools we already pay for
    

Changes:

- `backend-fastapi/app/main.py`: add `sentry_sdk.init(dsn=..., environment=..., traces_sample_rate=0.05)`
    
- `vispie_backend/settings.py`: uncomment + complete Sentry init
    
- `.env`, `.env.dev`: add `SENTRY_DSN_BACKEND`
    
- New Relic Alerts (via web console, no code):
    
- Policy: "FastAPI — critical"
    
- Condition 1: `Transaction error rate > 1%` over 5 min → notify Lark webhook
    
- Condition 2: `Transaction p99 > 10s` over 5 min → notify Lark webhook
    
- Policy: "Django — critical" (same two conditions)
    

Verification:

- Sentry: raise a synthetic exception via a one-off admin route → Sentry event appears <10 seconds
    
- NR: create a test alert (lower threshold temporarily, trigger, revert)
    

### 7.4 PR #4 — Destructive cleanup (delete what's dead)

**DO NOT MERGE UNTIL PR #2 HAS RUN STABLY FOR ≥1 WEEK IN PROD.**

Goals:

- Remove code, endpoints, schema, frontend that no longer serve purpose
    
- Reduce team cognitive load permanently
    

Changes:

- Delete frontend: `frontend-internal/src/routes/admin/vio-monitoring/` (entire subtree)
    
- Delete Sidebar link to `/admin/vio-monitoring/`
    
- Delete backend files:
    
- `app/api/monitoring/sla.py`
    
- `app/api/monitoring/analysis.py` (except `/cleanup` which we keep)
    
- `app/api/monitoring/overview.py` (overview card data)
    
- `app/api/monitoring/layers.py`
    
- `app/api/monitoring/agents.py`
    
- Rename file: `app/api/monitoring/probes.py` → `app/api/monitoring/synthetic.py`
    
- Rename table: `monitoring_probe_results` → `agent_synthetic_health` (Alembic)
    
- Drop tables (Alembic): `monitoring_sla_snapshots`, `monitoring_probe_configs`, `monitoring_alerts` (new alerter uses in-memory dedup)
    
- Delete scripts: `project_amy/host/modules/layer-health-probe-cron.sh`, `layer-health-probe.sh` (same for Vio)
    
- Keep: `e2e-synthetic-probe.py`, rename or keep as-is (doesn't matter for ops)
    

Verification:

- Re-run all frontend integration tests (none — which is itself a signal)
    
- Manual smoke: run one probe → row appears in `agent_synthetic_health`
    
- Run alert flow manually → Lark fires
    
- Inspect NR and Sentry dashboards — errors visible
    

---

## 8. Risks & Mitigations

```Plain
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **pg_cron alert path itself fails silently** | Medium | High | Add "heartbeat" check — the alerter writes a `last_ran` timestamp to a small table; separate watchdog alerts if not updated in 30 min |
| **Alert storm during real outage** (Amy down → every 15 min for hours) | High | Medium | 30-min dedup per (agent, platform, rule); optional escalation to `critical` after 2 hours |
| **Lark webhook down → alerts lost** | Low | High | Phase 2 (post-merge): add Slack webhook as fallback. Or integrate PagerDuty free tier as tertiary |
| **NR/Sentry outage → auxiliary channel dark** | Low | Medium | Primary channel (pg_cron → Lark) is independent. Auxiliary dark = annoying, not catastrophic |
| **Loss of historical SLA data** (if PR #4 drops snapshots table) | Low | Low | No client contract requires this. Snapshots table has 32 rows anyway. |
| **On-call process doesn't exist — alerts arrive but no one acts at 3am** | HIGH | HIGH | **This is a process issue, not a tech issue.** Flagged but out of scope for this design. Must be solved separately (on-call rotation, PagerDuty integration, etc.) |
```

---

## 9. Open Questions

1. **Do we need a tertiary backup alert channel** (Slack, PagerDuty) in case Lark webhook goes down? Owner to decide.
    
2. **On-call rotation**: who is paged at 3am Pacific when Amy breaks? This is out of scope for this doc but must be answered before PR #2 merge.
    
3. **Retention period: 7 days is the recommendation**. If team wants longer trailing view (e.g., "did it break this week" query needs 14 days), adjust the `retention_days` param in cleanup. Default 7 is safe for the alerting-first design.
    
4. **Owner of NR alert policies**: who has NR admin access to configure policies in PR #3? Currently assumed Zane.
    

---

## 10. Appendix A — Useful SQL

### 10.1 "Probe of probes" — the diagnostic query that started this redesign

```Plain
-- Three-layer alerting query — any row returned = alert
WITH freshness AS (
  SELECT 'stale_ingest' AS issue, agent_name, probe_type, ec2_source,
         MAX(probed_at) AS last_seen,
         EXTRACT(EPOCH FROM NOW() - MAX(probed_at))::int AS sec_stale
  FROM monitoring_probe_results
  WHERE probed_at >= NOW() - INTERVAL '1 day'
  GROUP BY agent_name, probe_type, ec2_source
  HAVING MAX(probed_at) < NOW() - INTERVAL '10 minutes'
),
flatline AS (
  SELECT 'all_timeout' AS issue, agent_name, probe_type,
         NULL::text AS ec2_source, NULL::timestamptz AS last_seen,
         COUNT(*)::int AS sec_stale
  FROM monitoring_probe_results
  WHERE probed_at >= NOW() - INTERVAL '1 hour'
  GROUP BY agent_name, probe_type
  HAVING COUNT(*) FILTER (WHERE status IN ('healthy','degraded')) = 0
     AND COUNT(*) > 3
),
volume_drop AS (
  SELECT 'volume_drop' AS issue, agent_name, probe_type, ec2_source,
         NULL::timestamptz, NULL::int
  FROM (
    SELECT agent_name, probe_type, ec2_source,
      COUNT(*) FILTER (WHERE probed_at >= NOW() - INTERVAL '1 hour') AS now_h,
      COUNT(*) FILTER (WHERE probed_at >= NOW() - INTERVAL '25 hours'
                        AND probed_at <  NOW() - INTERVAL '24 hours') AS yest_h
    FROM monitoring_probe_results
    WHERE probed_at >= NOW() - INTERVAL '25 hours'
    GROUP BY agent_name, probe_type, ec2_source
  ) x
  WHERE yest_h > 20 AND now_h < yest_h / 2
)
SELECT * FROM freshness
UNION ALL SELECT * FROM flatline
UNION ALL SELECT * FROM volume_drop
ORDER BY issue, agent_name;
```

### 10.2 Storage breakdown

```Plain
SELECT
  tablename,
  pg_size_pretty(pg_total_relation_size('public.'||tablename)) AS total,
  pg_size_pretty(pg_relation_size('public.'||tablename))       AS data,
  pg_size_pretty(pg_indexes_size('public.'||tablename))        AS indexes
FROM pg_tables
WHERE tablename LIKE 'monitoring_%'
ORDER BY pg_total_relation_size('public.'||tablename) DESC;
```

### 10.3 Index efficiency

```Plain
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size, idx_scan
FROM pg_stat_user_indexes
WHERE schemaname='public' AND relname='monitoring_probe_results'
ORDER BY idx_scan DESC;
```

---

## 11. Appendix B — Code Locations

```Plain
| Object | Path |
|--------|------|
| Probe writers | `project_amy/host/modules/e2e-synthetic-probe.py`, `project_amy/vio/infra/e2e-synthetic-probe.py` |
| FastAPI monitoring module | `backend-fastapi/app/api/monitoring/` (10 files, 1,605 lines total) |
| SQLAlchemy models | `common/database/models.py:6198–6290` (4 model classes) |
| Frontend pages | `frontend-internal/src/routes/admin/vio-monitoring/` (layout + 3 subroutes) |
| Frontend API client | `frontend-internal/src/lib/api/monitoring.ts` |
| Cron install scripts | `project_amy/host/modules/layer-health-probe-cron.sh`, `e2e-probe-cron.sh` |
```

---

## 12. Decision Log

```Plain
| Date | Decision | Status |
|------|----------|--------|
| 2026-04-14 | Option B' (precision-cut hybrid refactor) | Proposed |
| 2026-04-14 | Frontend to be fully deleted, not simplified | Proposed |
| 2026-04-14 | OpenTelemetry / Option D excluded (no SRE for 12 months) | Decided |
| 2026-04-14 | SLA report module to be deleted (no client contracts) | Decided |
| 2026-04-14 | Primary alert path bypasses New Relic | Proposed |
| TBD | Tertiary backup alert channel (Slack / PagerDuty) | Open |
| TBD | On-call rotation for off-hours alerts | Open |
```

---

_End of design document._