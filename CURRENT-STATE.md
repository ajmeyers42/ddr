# DDR Current State & Build Backlog

**As of**: September 2026  
**Phase**: Architecture complete → Phase 1 build starting

---

## What Exists Today

### Completed — In Repo

| Asset | Location | Status |
|-------|----------|--------|
| Agent system prompt v1.1 | `agent/system-prompt.md` | Complete — ready to deploy to Kibana Agent Builder |
| Tool inventory & in-cluster feature architecture | `architecture/tool-inventory-and-cluster-features.md` | Complete — design reference for all build tasks |
| esdiag repo gap analysis | `architecture/esdiag-repo-gap-analysis.md` | Complete — governs what to build vs. leverage |
| Deployment guide | `docs/deployment-guide.md` | Complete — step-by-step for Phase 1 and Phase 2 |
| Directory scaffold | All `elasticsearch/` and `kibana/` subdirs | Placeholder READMEs only — no implementation yet |

### Completed — Infrastructure

| Item | Status |
|------|--------|
| GitHub repo `ajmeyers42/ddr` | Public, live at https://github.com/ajmeyers42/ddr |
| Claude Project | Created, this conversation is scoped to it |
| Initial commit on `main` | Pushed — all architecture docs live |

---

## What Needs to Be Built

### Phase 1 — Kibana Space (No Owner Approval Needed)

These can be built and deployed immediately. They are space-isolated and do not touch esdiag's existing assets.

#### P1-1: Field Name Discovery (Blocker for most other tasks)
**Do this first.** All queries, transforms, and data views must use real esdiag field names.

```
GET _index_template/esdiag-*
```

Run against the hosted esdiag cluster. Document the field names for:
- `diagnostic.id`, `diagnostic.account`, `diagnostic.case`, `diagnostic.opportunity`, `diagnostic.user`
- Cluster health fields (status, shard counts, pending tasks)
- Node stats fields (JVM heap, CPU, disk, thread pools)
- Index stats fields (size, doc count, ILM phase, shard count)
- Slow query log fields (query shape, latency, index target)
- Ingest pipeline stats fields (throughput, failure count)

Output: `docs/esdiag-field-map.md` — a reference table of confirmed field names used by all subsequent build tasks.

---

#### P1-2: Kibana Space Setup
- Create Kibana space: name `DDR — Diagnostic Recommender`, URL identifier `ddr`
- Confirm space isolation from esdiag's default space

---

#### P1-3: Data Views (Phase 1 — pointing to existing esdiag indices)
Build and export as NDJSON to `kibana/data-views/`.

| Data View | Index Pattern | Notes |
|-----------|--------------|-------|
| `DDR — Diagnostics` | `esdiag-*` (confirm actual pattern from P1-1) | Primary view for all agent queries |
| `DDR — Cluster Health` | esdiag cluster health stream (confirm name) | For health trend dashboards |
| `DDR — Index Stats` | esdiag index stats stream (confirm name) | For capacity dashboards |
| `DDR — ILM Status` | esdiag ILM stream (confirm name) | For ILM compliance views |
| `DDR — Node Metrics` | esdiag node stats stream (confirm name) | For node resource dashboards |

All field names depend on P1-1 output.

---

#### P1-4: Advisory Dashboards
Build and export as NDJSON to `kibana/dashboards/`. Each dashboard uses DDR data views only.

**Dashboard 1 — Capacity & Planning**
Key panels:
- Storage used per cluster over time (area chart)
- Daily ingest rate trend (line chart)
- Disk headroom by node — days until watermark (gauge or bar)
- Index growth rate top-10 (table)
- ML forecast overlay (once Phase 2 ML is running — placeholder in Phase 1)

**Dashboard 2 — ILM & Data Tier Compliance**
Key panels:
- % of indices under ILM management per cluster (gauge)
- Indices by ILM phase (donut)
- Unmanaged index list (data table)
- Data tier distribution by GB: hot / warm / cold / frozen (stacked bar)
- ILM policy error log (table)

**Dashboard 3 — Findings History**
Key panels:
- Findings by severity over time (bar chart) — requires Phase 2 `recommender-findings-*` index
- Open vs. resolved findings (donut)
- Findings by category (table)
- Most recent findings (data table sorted by timestamp)
- Customer/account filter (use `diagnostic.account` field)

Note: Dashboard 3 needs the `recommender-findings-*` index from Phase 2. Build the shell in Phase 1, mark panels as "pending Phase 2" with placeholder text.

**Dashboard 4 — Multi-Cluster Comparison**
Key panels:
- Select two diagnostic IDs → compare key metrics side by side
- Version, shard count, heap %, disk %, ILM coverage (comparison table)
- Delta indicators (up/down arrows vs. baseline)

---

#### P1-5: Alerting Rules
Build and export to `kibana/alerts/`. Space-scoped — no cluster impact.

| Rule | Condition | Action |
|------|-----------|--------|
| Critical findings alert | `recommender-findings-*` contains CRITICAL severity document within last 1h | Webhook or email connector |
| ILM compliance drop | `ilm_coverage_pct` runtime field drops >10 points vs. previous diagnostic for same cluster | Flag for follow-up |
| Follow-up overdue | `recommender-followups-*` document past due date with status != resolved | Notify owner |

Note: The critical findings alert requires Phase 2 `recommender-findings-*`. Deploy the ILM and follow-up rules in Phase 1; the findings alert in Phase 2.

---

#### P1-6: Deploy Agent System Prompt
Deploy `agent/system-prompt.md` to Kibana Agent Builder within the DDR space.

Required before deploying:
- LLM connector configured (follow esdiag's LLM setup guide in `elastic/esdiag/docs/`)
- DDR Kibana space created (P1-2)
- At least the primary `DDR — Diagnostics` data view created (P1-3)

Before deploying, add three items to the prompt (per the gap analysis):
1. Reference to `.agents/skills/esdiag/` for collect/process workflows
2. Space context instruction (dashboard links must be `/s/ddr/` qualified)
3. Account metadata check instruction (flag bundles loaded without `diagnostic.account`)

---

### Phase 2 — Cluster-Level Additions (Require esdiag Owner Approval)

Open issues or PRs on `elastic/esdiag` before building any of these. Once approved, build definitions here and deploy.

#### P2-1: Audit Account Metadata Field Propagation
Run on the esdiag cluster:
```
GET esdiag-*/_mapping/field/diagnostic.account
```
If the field is absent from any sub-stream (Logstash, Kibana, ECK), open an issue on `elastic/esdiag` proposing a shared component template to enforce it across all data streams.

---

#### P2-2: Index Templates for Recommender Indices
Build and deploy to `elasticsearch/index-templates/`. Propose via esdiag owner before deploying.

| Template | Index Pattern | Key Fields |
|----------|--------------|------------|
| `recommender-findings` | `recommender-findings-*` | diagnostic.id, cluster.id, diagnostic.account, finding.category, finding.severity, finding.evidence, finding.recommendation, finding.capability_level, session.id, status, @timestamp |
| `recommender-projections` | `recommender-projections-*` | diagnostic.id, cluster.id, scenario (conservative/base/aggressive), storage_gb, node_count, monthly_cost_estimate, assumptions, @timestamp |
| `recommender-customer-context` | `recommender-customer-context-*` | account, contact, content (ELSER-indexed), source_type, @timestamp |

---

#### P2-3: Ingest Pipelines for Agent Output
Build and deploy to `elasticsearch/ingest-pipelines/`.

| Pipeline | Purpose |
|----------|---------|
| `recommender-findings-formatter` | Validates required fields, sets default status to `open`, generates session ID if absent |
| `recommender-customer-context-enrichment` | Adds cluster and account metadata; triggers ELSER inference pipeline |

---

#### P2-4: Transforms
Build definitions in `elasticsearch/transforms/`. Use real field names from P1-1.

| Transform | Source | Output | Refresh |
|-----------|--------|--------|---------|
| `recommender-cluster-health-summary` | esdiag cluster health stream | `recommender-cluster-health-*` | Continuous |
| `recommender-node-metrics` | esdiag node stats stream | `recommender-node-metrics-*` | Continuous |
| `recommender-index-growth-rate` | esdiag index stats stream | `recommender-index-stats-*` | Continuous |
| `recommender-ilm-compliance` | esdiag ILM stream | `recommender-ilm-status-*` | Continuous |
| `recommender-shard-distribution` | esdiag shard stats stream | `recommender-shard-dist-*` | Continuous |
| `recommender-slow-query-aggregation` | esdiag slow query stream | `recommender-slow-queries-*` | Continuous |
| `recommender-custom-code-inventory` | esdiag pipeline/index stats streams | `recommender-custom-code-*` | Scheduled |
| `recommender-current-cluster-state` (latest) | esdiag cluster health stream | `recommender-current-state-*` | Continuous |

---

#### P2-5: ML Anomaly Detection Jobs
Build definitions in `elasticsearch/ml-jobs/`. Require Platinum subscription.

| Job | Detector | Partition |
|-----|----------|-----------|
| `recommender-jvm-gc-pressure` | mean(gc_old_gen_collection_count) | per node |
| `recommender-ingest-rate-drop` | low_mean(indexing_rate) | per cluster |
| `recommender-search-latency-spike` | high_mean(search_latency_p99) | per node |
| `recommender-disk-growth-acceleration` | high_non_zero_count(disk_used_pct) | per node |
| `recommender-bulk-rejection-rate` | high_count(bulk_rejections) | per node |
| `recommender-shard-count-growth` | high_mean(total_shard_count) | per cluster |
| `recommender-master-instability` | high_count(master_elections) | per cluster |

---

#### P2-6: ML Forecasting Jobs
| Job | Target metric | Horizons |
|-----|--------------|---------|
| `recommender-storage-forecast` | total disk used per cluster | 30d, 90d, 365d |
| `recommender-ingest-rate-forecast` | daily ingest GB per cluster | 30d, 90d |
| `recommender-search-load-forecast` | searches/sec per cluster | 30d, 90d |

---

#### P2-7: ELSER Semantic Index
Configure ELSER inference pipeline against `recommender-customer-context-*` and `recommender-findings-*` to enable semantic similarity search — the `find_similar_clusters` tool depends on this.

---

### Phase 3 — Longer Term

| Item | Description |
|------|-------------|
| MCP endpoint | A new MCP server exposing DDR tools for external agent access (Claude, Cursor, other frameworks). Allows analysis work to be split between in-cluster and external agents. Not yet designed. |
| Contribution to esdiag | Capacity planning and feature adoption dashboards as upstream contributions to `elastic/esdiag`. Once DDR dashboards are validated, propose them as PRs. |
| esdiag `setup --profile recommender` | Propose a new setup profile to esdiag owner that installs all Phase 2 DDR cluster assets in one command, managed under the esdiag repo lifecycle. |
| Agent prompt v2 | After real diagnostic sessions produce output, iterate on the prompt based on what works and what doesn't. Expected: Mode 2 capability table needs tuning, capacity projection assumptions need calibration from real data. |

---

## Task Sequencing for the Next Agent Session

If you are an agent picking this up, do these in order:

1. Read `HANDOFF.md` for full context
2. Read `agent/system-prompt.md` to understand what the agent does
3. Read `architecture/esdiag-repo-gap-analysis.md` for the esdiag relationship
4. Your first build task: **P1-1 field name discovery** — run `GET _index_template/esdiag-*` and produce `docs/esdiag-field-map.md`
5. With field names in hand: **P1-3 data views** — build the DDR data views for the Kibana space
6. Then: **P1-4 dashboards** — Capacity & Planning and ILM Compliance first (don't require Phase 2 indices)
7. Then: **P1-6 agent deployment** — update system prompt with the three additions noted above, deploy to Kibana Agent Builder
8. Run a test session with a real or sample diagnostic bundle to validate agent output
9. Open issues on `elastic/esdiag` for Phase 2 coordination

---

*Document version: 1.0*  
*Repo: https://github.com/ajmeyers42/ddr*
