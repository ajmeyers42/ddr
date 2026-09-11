# Dashboards

Advisory dashboards deployed into the DDR Kibana space. These complement esdiag's existing support/troubleshooting dashboards — they are not replacements.

**Status**: Not yet built. Phase 1 priority.

## Planned dashboards

| Dashboard | Purpose |
|-----------|---------|
| Capacity & Planning | 12-month storage/compute growth with ML forecast overlay |
| Feature Adoption Assessment | What capabilities the cluster has vs. what it uses |
| Findings History | All persisted agent findings and their status |
| Multi-Cluster Comparison | Side-by-side metrics across two or more diagnostics |
| ML Anomaly Summary | Anomaly scores and influencers across all ML jobs |

## Building dashboards

Before building, read the actual esdiag index template field names:
```
GET _index_template/esdiag-*
```

Use the DDR data views (not esdiag's originals) as the data source for all DDR dashboards.

Export from Kibana as NDJSON (Stack Management → Saved Objects → Export with dependencies).
