# Ingest Pipelines

Pipelines for structuring and validating agent output before it is indexed into recommender indices.

These pipelines are for **new recommender indices only** — they do not modify esdiag's existing ingest pipelines.

**Status**: Not yet built.

## Planned pipelines

| Pipeline ID | Purpose |
|-------------|---------|
| `recommender-findings-formatter` | Validates and normalizes agent findings before indexing into `recommender-findings-*` |
| `recommender-customer-context-enrichment` | Adds cluster and account metadata to customer context documents |

## Deployment requirement

Cluster-level access required. Coordinate with esdiag repo owner before deploying.
