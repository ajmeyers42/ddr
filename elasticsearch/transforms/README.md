# Transforms

Pre-aggregation transforms that run against existing esdiag data streams and write to `recommender-*` output indices.

**Status**: Not yet built — requires field name mapping from deployed esdiag index templates first.

## Planned transforms

| Transform ID | Reads from | Writes to | Purpose |
|-------------|-----------|-----------|---------|
| `recommender-cluster-health-summary` | `esdiag-*` | `recommender-cluster-health-*` | Daily health rollup per cluster |
| `recommender-node-metrics` | `esdiag-*` | `recommender-node-metrics-*` | Per-node resource utilization |
| `recommender-index-growth-rate` | `esdiag-*` | `recommender-index-stats-*` | Per-index size/doc growth rate |
| `recommender-ilm-compliance` | `esdiag-*` | `recommender-ilm-status-*` | ILM policy coverage and phase compliance |
| `recommender-shard-distribution` | `esdiag-*` | `recommender-shard-dist-*` | Shard allocation per node and tier |
| `recommender-slow-query-aggregation` | `esdiag-*` | `recommender-slow-queries-*` | Top-N slow query shapes |
| `recommender-custom-code-inventory` | `esdiag-*` | `recommender-custom-code-*` | Custom scripts and external dependencies |

## Next step

Run `GET _index_template/esdiag-*` against the esdiag cluster and update this directory with transform definitions using real field names.

## Deployment requirement

Cluster-level access required. Coordinate with esdiag repo owner before deploying. See `architecture/esdiag-repo-gap-analysis.md`.
