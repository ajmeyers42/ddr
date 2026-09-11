# DDR Deployment Guide

## Prerequisites

- Access to the hosted esdiag cluster (Kibana and Elasticsearch endpoints)
- An API key with sufficient privileges to create Kibana saved objects and read esdiag data streams
- A separate API key (or the same, with appropriate privileges) for Phase 2 cluster-level assets
- The esdiag cluster must be running esdiag v0.13 or later (Kibana assets installed)

## Phase 1 — Kibana Space Deployment

Phase 1 requires only Kibana saved object privileges. No cluster-level changes.

### 1. Create the DDR Kibana Space

In Kibana → Stack Management → Spaces → Create Space:
- Name: `DDR — Diagnostic Recommender`
- URL identifier: `ddr`
- Disable features not needed (optional): Maps, APM, etc.

### 2. Create Data Views

For each data view in `kibana/data-views/`, import via:

```
POST kbn:/s/ddr/api/data_views/data_view
```

Or use Stack Management → Data Views within the DDR space.

Data views point to existing esdiag index patterns. No new indices are required for Phase 1.

**Required first step**: Run the following to confirm the actual esdiag index patterns and field names in your cluster before creating data views:

```esql
GET _index_template/esdiag-*
```

Update data view index patterns to match what your esdiag deployment actually uses.

### 3. Import Dashboards

Import each dashboard NDJSON from `kibana/dashboards/` via:

```
POST kbn:/s/ddr/api/saved_objects/_import?overwrite=false
```

Or use Stack Management → Saved Objects → Import within the DDR space.

### 4. Deploy Alerting Rules

Import alerting rules from `kibana/alerts/` via the Kibana Alerting API within the DDR space.

### 5. Deploy the Agent

The DDR agent system prompt is in `agent/system-prompt.md`.

Deploy to Kibana Agent Builder (Elastic AI Assistant configuration) within the DDR space. The agent requires:
- Read access to all esdiag data streams
- Write access to `recommender-findings-*` and `recommender-projections-*` (Phase 2)
- An LLM connector configured (see the esdiag LLM setup guide in the esdiag repo)

---

## Phase 2 — Cluster-Level Additions

Phase 2 assets require cluster-level privileges and **coordination with the esdiag repo owner** before deployment. Do not deploy these without approval.

See `architecture/esdiag-repo-gap-analysis.md` → Category B and C sections for the specific changes to propose.

### Proposed additions requiring owner approval

1. Component template for account metadata field propagation across all esdiag sub-streams
2. Transforms (see `elasticsearch/transforms/`)
3. ML anomaly detection and forecasting jobs (see `elasticsearch/ml-jobs/`)
4. ELSER semantic index configuration
5. New index templates for recommender indices (see `elasticsearch/index-templates/`)
6. Ingest pipelines for agent output (see `elasticsearch/ingest-pipelines/`)

### Deployment order (once approved)

```
# 1. Index templates first (defines the mapping before data arrives)
PUT _index_template/recommender-findings
PUT _index_template/recommender-projections
PUT _index_template/recommender-customer-context

# 2. Ingest pipelines
PUT _ingest/pipeline/recommender-findings-formatter
PUT _ingest/pipeline/recommender-customer-context-enrichment

# 3. Transforms (reads esdiag indices, writes to recommender-* indices)
PUT _transform/recommender-cluster-health-summary
PUT _transform/recommender-node-metrics
PUT _transform/recommender-index-growth-rate
PUT _transform/recommender-ilm-compliance
PUT _transform/recommender-shard-distribution
PUT _transform/recommender-slow-query-aggregation
PUT _transform/recommender-custom-code-inventory

# Start transforms
POST _transform/recommender-*/_start

# 4. ML jobs
PUT _ml/anomaly_detectors/recommender-jvm-gc-pressure
PUT _ml/anomaly_detectors/recommender-ingest-rate-drop
PUT _ml/anomaly_detectors/recommender-search-latency-spike
PUT _ml/anomaly_detectors/recommender-disk-growth-acceleration
PUT _ml/anomaly_detectors/recommender-bulk-rejection-rate
PUT _ml/anomaly_detectors/recommender-shard-count-growth
PUT _ml/anomaly_detectors/recommender-master-instability

PUT _ml/datafeeds/recommender-*

# Open and start jobs
POST _ml/anomaly_detectors/recommender-*/_open
POST _ml/datafeeds/recommender-*/_start
```

---

## Verification

After Phase 1:
```esql
# Confirm esdiag data is reachable from DDR space
GET /esdiag-*/_count

# Confirm DDR data views are configured correctly
GET kbn:/s/ddr/api/data_views
```

After Phase 2:
```esql
# Confirm transforms are running
GET _transform/recommender-*/_stats

# Confirm ML jobs are open
GET _ml/anomaly_detectors/recommender-*
```

---

## Rollback

Phase 1 (Kibana-only): Delete the DDR space. No cluster impact.

Phase 2: Stop and delete transforms and ML jobs. Delete recommender-* indices. No impact on esdiag data.
