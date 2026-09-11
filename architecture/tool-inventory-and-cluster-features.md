# Diagnostic-Driven Recommender Agent
## Tool Inventory & In-Cluster Feature Architecture

---

## Overview

This document defines the full tool set and supporting Elastic in-cluster feature stack for the Diagnostic-Driven Recommender Agent. It is structured in two layers:

- **Agent Tools** — the callable functions the agent invokes to retrieve, compare, query, and persist data during a session
- **In-Cluster Features** — the Elasticsearch and Kibana components that produce, store, and expose the data those tools access

Every tool maps to one or more in-cluster features. Every in-cluster feature should have a corresponding tool that makes it accessible to the agent.

---

## Agent Tool Inventory

### Category 1 — Diagnostic Retrieval

Tools for locating and loading diagnostic bundle data from the indexed diagnostic store.

| Tool | Description | Primary Index / Source |
|------|-------------|----------------------|
| `list_diagnostics` | Enumerate available diagnostics by customer, cluster ID, date range, or deployment type. Returns bundle IDs, timestamps, version, and deployment type. | `diag-bundles-*` |
| `get_diagnostic` | Retrieve a full or partial diagnostic record by bundle ID. Supports field-level projection to avoid loading unnecessary data. | `diag-bundles-*` |
| `search_diagnostics` | ES\|QL or DSL search across all indexed diagnostic data. Supports filtering by version, tier, deployment type, ILM state, shard count, etc. | `diag-bundles-*` |
| `compare_diagnostics` | Diff two diagnostic bundles (same cluster over time, or two clusters). Returns delta on key fields: shard count, JVM pressure, index sizes, ILM state, version. | `diag-bundles-*`, `diag-cluster-metrics-*` |
| `get_diagnostic_context` | Retrieve customer conversation notes, prior estimates, architecture docs, or prior findings associated with a diagnostic or customer ID. | `diag-customer-context-*` |

---

### Category 2 — Cluster Metrics & State

Tools for querying pre-aggregated cluster health, node, and index metrics produced by transforms.

| Tool | Description | Primary Index / Source |
|------|-------------|----------------------|
| `get_cluster_health_summary` | Pull the latest or historical rollup of cluster health indicators: status, unassigned shards, pending tasks, GC pressure, circuit breaker trips. | `diag-cluster-health-*` (transform output) |
| `get_node_metrics` | Retrieve node-level resource utilization: JVM heap %, CPU %, disk used/free, thread pool queue depths, indexing and search rates. | `diag-node-metrics-*` (transform output) |
| `get_index_stats` | Pull per-index statistics: primary/replica size, doc count, shard count, ILM phase, age, segment count, refresh/flush intervals. | `diag-index-stats-*` (transform output) |
| `get_ilm_status` | Query ILM policy compliance across indices: which indices are managed, current phase, time in phase, policy errors, unmanaged index count. | `diag-ilm-status-*` (transform output) |
| `get_shard_distribution` | Retrieve shard allocation by node, role, and index: shard counts per node, shard sizes, replication factor, hot/warm/cold tier distribution. | `diag-shard-dist-*` (transform output) |
| `get_slow_query_summary` | Return the top-N query shapes by frequency and latency from aggregated slow query log analysis. Includes query fingerprints and index patterns. | `diag-slow-queries-*` (transform output) |
| `get_ingest_pipeline_stats` | Return per-pipeline processor stats: throughput, failure rates, processor latency, bulk reject counts. | `diag-ingest-stats-*` (transform output) |

---

### Category 3 — ML & Anomaly Detection

Tools for querying machine learning job results, forecasts, and running semantic inference.

| Tool | Description | Primary Index / Source |
|------|-------------|----------------------|
| `get_anomaly_results` | Retrieve ML anomaly detection results for a cluster and time range. Returns anomaly scores, influencers, and the metric that triggered. Filter by job group (JVM, ingest, search, disk, shards). | `.ml-anomalies-*` |
| `get_forecasts` | Pull ML forecast outputs for storage growth, ingest rate, or search load. Returns point estimates and confidence intervals at 30, 90, and 365 days. | `.ml-forecast-*` |
| `find_similar_clusters` | Semantic search over historical diagnostic findings using ELSER. Given a current diagnostic's key characteristics, return the most similar past diagnostics with their findings and outcomes. Useful for pattern matching and prior recommendation lookup. | `diag-findings-*` (ELSER-indexed) |
| `search_findings_history` | Full-text and semantic search over prior agent output — past findings, recommendations, and their status. Scoped by customer, cluster, date, or finding category. | `diag-findings-*` |
| `classify_finding_severity` | Run a trained classification model against a finding description to suggest severity level (CRITICAL / HIGH / MEDIUM / OPPORTUNITY). Avoids inconsistent severity assignment across sessions. | Trained model (Platinum/Enterprise) |

---

### Category 4 — Knowledge & Documentation

Tools for accessing Elastic product knowledge, version compatibility data, and deprecation information.

| Tool | Description | Primary Index / Source |
|------|-------------|----------------------|
| `search_elastic_docs` | Semantic search against indexed Elastic documentation. Version-scoped: results are filtered to docs relevant to the cluster's current version and target version if migration is in scope. | `elastic-docs-*` (ELSER-indexed) or Elastic Docs MCP |
| `get_version_matrix` | Return compatibility and deprecation data for a given version pair: deprecated APIs, removed features, behavior changes, required re-index operations. | `diag-version-matrix` (static, periodically updated) |
| `get_feature_availability` | Given a feature name and subscription level, return whether it is available, what version it requires, and what configuration is needed to enable it. | `diag-feature-registry` (static, periodically updated) |

---

### Category 5 — Visualization & Dashboard Access

Tools for surfacing pre-built Kibana dashboards and data views relevant to the current analysis.

| Tool | Description | Primary Index / Source |
|------|-------------|----------------------|
| `list_dashboards` | Enumerate available diagnostic dashboards by category (health, ingest, search, capacity, ML). Returns dashboard IDs, titles, and applicable diagnostic types. | Kibana saved objects API |
| `get_dashboard_url` | Generate a deep-link URL to a specific Kibana dashboard, optionally pre-filtered to a cluster ID, time range, or node. | Kibana saved objects + query params |
| `get_data_view` | Retrieve a data view definition by name or ID — field mappings, runtime fields, index pattern. Useful for explaining what data is available for a given category of analysis. | Kibana saved objects API |
| `run_esql_query` | Execute an arbitrary ES\|QL query against the diagnostic indices and return results. For ad-hoc analysis during a session when pre-built aggregations don't cover the need. | Elasticsearch ES\|QL endpoint |

---

### Category 6 — Output & Persistence

Tools for persisting analysis outputs, creating reports, and triggering follow-on actions.

| Tool | Description | Primary Index / Source |
|------|-------------|----------------------|
| `write_findings` | Persist structured analysis findings to the findings index. Schema includes: cluster ID, diagnostic ID, customer ID, session ID, finding category, severity, evidence, recommendation, capability mapping (subscription level), and status. | `diag-findings-*` |
| `write_capacity_projection` | Persist a capacity and cost projection to the projections index. Includes baseline metrics, assumptions, and scenario outputs (conservative/base/aggressive). | `diag-projections-*` |
| `create_summary_report` | Generate a structured markdown or HTML summary report from the current session's findings, formatted for delivery to a customer or account team. | Agent output → `diag-reports-*` |
| `flag_for_follow_up` | Mark a finding or projection for follow-up review, with a due date and owner. Triggers a Kibana alert if the follow-up date passes without a status update. | `diag-followups-*` |

---

## In-Cluster Feature Architecture

### Ingest Layer — Getting Diagnostic Data In

Before any analysis is possible, raw diagnostic bundle output must be parsed, normalized, and enriched as it enters the cluster.

**Ingest Pipelines**

| Pipeline | Purpose |
|----------|---------|
| `diag-bundle-parser` | Parses raw diagnostic tool output (JSON/zip) into structured Elasticsearch documents. Extracts cluster health, node stats, index stats, ILM policies, mapping info, slow logs, and security configuration. Produces one document per logical entity (cluster, node, index, policy) rather than one blob per bundle. |
| `diag-enrichment` | Enriches parsed documents with customer metadata (customer ID, account team, deployment type label), version classification (EOL flag, upgrade target), and subscription level. |
| `diag-slow-query-normalizer` | Normalizes slow query log entries: extracts query shape, generates a stable fingerprint hash (stripping literal values for aggregation), tags the query type (search, aggregation, knn, etc.). |
| `diag-findings-formatter` | Structures agent output (findings, recommendations, projections) for consistent persistence. Applies the canonical findings schema and generates a stable session ID. |

---

### Transform Layer — Pre-Aggregating for Fast Agent Access

Rather than requiring the agent to perform heavy aggregations at query time, transforms produce pre-computed summaries that agent tools can access cheaply.

**Pivot Transforms** (aggregate across documents, scheduled refresh)

| Transform | Output Index | Description |
|-----------|-------------|-------------|
| `diag-cluster-health-summary` | `diag-cluster-health-*` | Daily rollup per cluster: status history, unassigned shard count, pending task count, GC pause frequency/duration, circuit breaker trip count. Enables health trend analysis without re-scanning the full bundle. |
| `diag-node-resource-summary` | `diag-node-metrics-*` | Per-node, per-day aggregation: avg/max JVM heap %, CPU %, disk used %, thread pool rejections, indexing rate, search rate. Sized for 90-day retention to support trend and capacity analysis. |
| `diag-index-growth-rate` | `diag-index-stats-*` | Per-index storage size and doc count over time. Calculates rolling 7-day and 30-day growth rate. Feeds directly into the capacity projection model. |
| `diag-ilm-compliance` | `diag-ilm-status-*` | Per-cluster ILM compliance snapshot: total managed indices, unmanaged count, indices stuck in phase, policy error count, % of data under each ILM phase. |
| `diag-shard-distribution` | `diag-shard-dist-*` | Shard allocation per node and per tier. Calculates shard-to-node ratio, identifies over-sharded nodes, flags indices where shard size falls below 10GB (over-sharded) or exceeds 50GB (under-sharded). |
| `diag-slow-query-aggregation` | `diag-slow-queries-*` | Aggregates slow query log entries by fingerprint hash. Produces top-N query shapes by frequency, P99 latency, and index target. Enables the agent to identify query anti-patterns without parsing raw log entries. |
| `diag-ingest-pipeline-perf` | `diag-ingest-stats-*` | Per-pipeline, per-day aggregation of processor throughput, failure rates, and latency. Identifies which pipelines are under stress and where failures occur. |
| `diag-custom-code-inventory` | `diag-custom-code-*` | Identifies presence of custom scripts (Painless in ingest pipelines, Watcher), external schedulers referenced in configurations, and non-standard ingest patterns. Feeds the "replace with native capability" recommendation path. |

**Latest Transforms** (current state snapshot, continuous or near-real-time)

| Transform | Output Index | Description |
|-----------|-------------|-------------|
| `diag-current-cluster-state` | `diag-current-state-*` | Most recent record of all key cluster health indicators per cluster ID. Used by the agent's session opening to give an immediate current-state summary without querying historical data. |
| `diag-current-ilm-state` | `diag-current-ilm-*` | Latest ILM phase per index per cluster. Enables fast "is ILM working?" checks at session start. |
| `diag-current-node-state` | `diag-current-nodes-*` | Latest resource readings per node. Enables fast "is any node in distress?" checks. |

---

### ML Layer — Anomaly Detection, Forecasting, and Semantic Search

**Anomaly Detection Jobs**

Each job runs against the transformed (not raw) diagnostic indices. All jobs output to `.ml-anomalies-*` and are surfaced via the `get_anomaly_results` tool.

| Job | Metric | Why It Matters |
|-----|--------|---------------|
| `diag-jvm-gc-pressure` | Old gen GC frequency and duration per node | Sustained GC pressure precedes heap exhaustion and node instability. Catches gradual degradation before it becomes an incident. |
| `diag-ingest-rate-drop` | Indexing rate per cluster | Sudden drops indicate upstream pipeline failure, rejected bulks, or source connectivity issues the cluster owner may not have visibility into. |
| `diag-search-latency-spike` | P99 search latency per node | Catches query performance regressions caused by mapping changes, new query patterns, or resource contention. |
| `diag-disk-growth-acceleration` | Rate of change of disk used % per node | Detects when disk growth is accelerating — not just that disk is filling, but that it is filling faster than it was. Feeds the capacity projection model with a real observed growth rate. |
| `diag-bulk-rejection-rate` | Bulk rejection count per node | Rejection spikes indicate indexing queue saturation — typically caused by under-provisioned thread pools or ILM rollover misconfiguration. |
| `diag-shard-count-growth` | Total shard count per cluster | Uncontrolled shard proliferation is one of the most common self-inflicted cluster performance issues. Anomalous growth indicates missing or misconfigured ILM. |
| `diag-master-instability` | Master election frequency | Frequent elections indicate master node instability — network issues, resource pressure, or split-brain risk. |

**ML Forecasting Jobs**

| Job | Forecast Horizon | Use |
|-----|-----------------|-----|
| `diag-storage-forecast` | 30 / 90 / 365 days | Primary input for the capacity projection model. Provides ML-derived growth rate that can validate or replace the agent's assumed growth rate. |
| `diag-ingest-rate-forecast` | 30 / 90 days | Projects future indexing volume. Used to size compute (thread pools, node count) alongside storage. |
| `diag-search-load-forecast` | 30 / 90 days | Projects search request volume. Used to validate whether current node count will sustain query SLAs at projected growth. |

**NLP / Semantic Search (ELSER)**

| Feature | Index | Use |
|---------|-------|-----|
| Findings semantic index | `diag-findings-*` | All persisted findings are ELSER-encoded. Enables `find_similar_clusters` — given a current cluster's characteristics, find the most semantically similar past analyses and their outcomes. Prevents duplicate work and surfaces proven recommendations. |
| Documentation semantic index | `elastic-docs-*` | Elastic documentation is ELSER-encoded and version-tagged. The `search_elastic_docs` tool performs semantic search to retrieve the most relevant guidance for a finding, without requiring the agent to know exact document titles or URLs. |
| Customer context index | `diag-customer-context-*` | Customer notes, conversation summaries, and prior estimates are ELSER-encoded. Enables semantic retrieval of relevant account history during a session — e.g., finding a prior sizing conversation when building a new capacity projection. |

---

### Data Views

One data view per logical domain. Each is the canonical entry point for Kibana Discover exploration and Lens visualization of that domain.

| Data View | Index Pattern | Description |
|-----------|--------------|-------------|
| `Diagnostic — Bundles` | `diag-bundles-*` | Raw parsed diagnostic bundle data. Cluster, node, index, ILM, mapping, and security entities. |
| `Diagnostic — Cluster Health` | `diag-cluster-health-*` | Transform output: cluster health trends over time. |
| `Diagnostic — Node Metrics` | `diag-node-metrics-*` | Transform output: per-node resource utilization trends. |
| `Diagnostic — Index Stats` | `diag-index-stats-*` | Transform output: per-index size, growth, shard, and ILM data. |
| `Diagnostic — ILM Compliance` | `diag-ilm-status-*` | Transform output: ILM policy coverage and phase compliance. |
| `Diagnostic — Shard Distribution` | `diag-shard-dist-*` | Transform output: shard allocation and sizing by node and tier. |
| `Diagnostic — Slow Queries` | `diag-slow-queries-*` | Transform output: aggregated slow query shapes and latency. |
| `Diagnostic — Ingest Performance` | `diag-ingest-stats-*` | Transform output: pipeline throughput, failures, and processor latency. |
| `Diagnostic — ML Results` | `.ml-anomalies-*`, `.ml-forecast-*` | Anomaly detection results and forecast outputs across all ML jobs. |
| `Diagnostic — Findings` | `diag-findings-*` | Agent output: persisted findings, recommendations, and their status. |
| `Diagnostic — Projections` | `diag-projections-*` | Agent output: capacity and cost projections. |
| `Diagnostic — Customer Context` | `diag-customer-context-*` | Customer notes, conversation history, prior estimates. |

**Runtime Fields** (defined at the data view level, available across dashboards and tools)

| Field | Calculation | Use |
|-------|-------------|-----|
| `shard_to_node_ratio` | `total_shard_count / data_node_count` | Over-sharding indicator. Flags when ratio exceeds 1000 per node (Elastic guideline). |
| `heap_usage_pct` | `jvm_heap_used_bytes / jvm_heap_max_bytes × 100` | Normalized heap pressure across nodes regardless of heap size. |
| `disk_headroom_days` | `(disk_free_bytes / daily_growth_bytes)` | Days until the low watermark is hit at current growth rate. Key input to urgency ranking. |
| `version_age_days` | `today - version_release_date` | Identifies how far behind the cluster is. Flags when a minor or major version exceeds a configurable staleness threshold. |
| `ilm_coverage_pct` | `managed_index_count / total_index_count × 100` | What percentage of the cluster's indices are under ILM management. |
| `effective_replication_factor` | `total_shard_count / primary_shard_count` | Distinguishes configured replication from actual replication (accounts for unassigned replicas). |

---

### Dashboards

Dashboards are reference views — the agent can link to them, but they also stand alone as operational tools for the customer's team.

| Dashboard | Purpose | Key Panels |
|-----------|---------|-----------|
| Cluster Health Overview | At-a-glance health across all indexed diagnostics. Filters by customer, cluster ID, or date range. | Health status timeline, unassigned shards, GC pressure heatmap, disk headroom by node, circuit breaker trip frequency |
| Node Resource Utilization | Per-node resource deep-dive. Side-by-side comparison of heap, CPU, disk, and thread pool state. | JVM heap trend per node, disk growth per node, indexing/search rate by node, thread pool queue depth |
| Ingest Pipeline Health | Throughput and failure analysis across ingest pipelines, Logstash, and Elastic Agent. | Pipeline throughput trend, bulk rejection rate, processor failure by pipeline, backpressure indicators |
| Search Performance | Query latency and cache efficiency. Identifies slow query patterns by shape. | P50/P99/P999 latency trend, cache hit rates (request/field/query), top slow query shapes by fingerprint |
| Data Tier & ILM Compliance | Visualizes data distribution across hot/warm/cold/frozen and ILM adherence. | Tier distribution by GB and index count, ILM phase timeline per index, unmanaged index list, policy error log |
| Capacity Trend & Projection | Storage and compute growth with ML-derived forecasts overlaid. | Storage growth per cluster (actual + ML forecast), ingest rate trend + forecast, node count headroom, disk headroom gauge |
| ML Anomaly Results | Anomaly scores and influencers across all ML jobs. | Anomaly score timeline by job, influencer breakdown, anomaly severity heatmap by cluster, forecast confidence bands |
| Multi-Cluster Comparison | Side-by-side comparison of two or more diagnostics — useful for before/after analysis or peer cluster review. | Metric comparison table (version, shard count, heap %, disk %, ILM coverage), delta indicators |
| Findings History | All persisted agent findings and their status. | Findings by severity over time, open vs resolved, findings by category, recommendation status tracker |
| Custom Code Inventory | Surfaces custom scripts, external dependencies, and non-native patterns detected across diagnostics. | Custom Painless scripts by cluster, external scheduler references, non-ILM retention patterns, Logstash pipelines that overlap with native ingest capabilities |

---

### Alerting

Alerts that make the agent proactive rather than purely reactive.

| Alert | Condition | Action |
|-------|-----------|--------|
| New diagnostic with critical findings | A diagnostic is ingested and `write_findings` produces one or more CRITICAL-severity findings | Notify the account team channel with finding summary and deep-link to the Findings History dashboard |
| ML anomaly threshold exceeded | Any ML job produces an anomaly score above a configurable threshold (default: 75) for a cluster with an active customer context record | Flag for follow-up review; optionally trigger an agent session pre-loaded with the anomaly context |
| Storage forecast crosses capacity threshold | The `diag-storage-forecast` job projects disk full within 90 days at current growth rate | Trigger a capacity projection session for the affected cluster |
| ILM compliance degradation | `ilm_coverage_pct` drops more than 10 points since the last diagnostic for the same cluster | Flag as a regression — ILM policies may have been bypassed or new indices are being created outside templates |
| Follow-up overdue | A `flag_for_follow_up` record has passed its due date without a status update | Notify the owner and escalate to account team |

---

## Tool-to-Feature Mapping Summary

```
Agent Tool                     → In-Cluster Feature(s)
─────────────────────────────────────────────────────
list_diagnostics               → diag-bundles-* (data view)
get_diagnostic                 → diag-bundles-* (data view)
compare_diagnostics            → diag-bundles-* + diag-cluster-metrics-*
get_cluster_health_summary     → diag-cluster-health-summary (transform)
get_node_metrics               → diag-node-resource-summary (transform)
get_index_stats                → diag-index-growth-rate (transform)
get_ilm_status                 → diag-ilm-compliance (transform)
get_shard_distribution         → diag-shard-distribution (transform)
get_slow_query_summary         → diag-slow-query-aggregation (transform)
get_ingest_pipeline_stats      → diag-ingest-pipeline-perf (transform)
get_anomaly_results            → diag-jvm-gc-pressure, -ingest-rate-drop,
                                 -search-latency-spike, -disk-growth, etc. (ML jobs)
get_forecasts                  → diag-storage-forecast, -ingest-rate, -search-load (ML)
find_similar_clusters          → diag-findings-* (ELSER semantic index)
search_findings_history        → diag-findings-* (ELSER semantic index)
search_elastic_docs            → elastic-docs-* (ELSER) or Elastic Docs MCP
get_version_matrix             → diag-version-matrix (static reference index)
get_feature_availability       → diag-feature-registry (static reference index)
get_dashboard_url              → Kibana saved objects
run_esql_query                 → Elasticsearch ES|QL
write_findings                 → diag-findings-* + diag-findings-formatter pipeline
write_capacity_projection      → diag-projections-*
create_summary_report          → Agent output → diag-reports-*
flag_for_follow_up             → diag-followups-* → Kibana alerting
```

---

*Document version: 1.0*
*Agent: Diagnostic-Driven Recommender Agent*
*Scope: ECH · Self-managed · ECK · ECE · Multi-cluster*
