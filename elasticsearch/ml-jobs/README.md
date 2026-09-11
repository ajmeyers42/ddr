# ML Jobs

Anomaly detection and forecasting jobs against esdiag data streams.

**Status**: Not yet built — requires field name mapping and transform outputs first.

## Planned anomaly detection jobs

| Job ID | Metric | Purpose |
|--------|--------|---------|
| `recommender-jvm-gc-pressure` | Old gen GC frequency/duration per node | Catch heap exhaustion before it happens |
| `recommender-ingest-rate-drop` | Indexing rate per cluster | Detect upstream pipeline failures |
| `recommender-search-latency-spike` | P99 search latency per node | Catch query performance regressions |
| `recommender-disk-growth-acceleration` | Rate of change of disk used % | Detect accelerating disk growth |
| `recommender-bulk-rejection-rate` | Bulk rejection count per node | Flag indexing queue saturation |
| `recommender-shard-count-growth` | Total shard count per cluster | Detect uncontrolled shard proliferation |
| `recommender-master-instability` | Master election frequency | Flag master node instability |

## Planned forecasting jobs

| Job ID | Horizon | Purpose |
|--------|---------|---------|
| `recommender-storage-forecast` | 30/90/365 days | Primary input to capacity projection |
| `recommender-ingest-rate-forecast` | 30/90 days | Project future compute requirements |
| `recommender-search-load-forecast` | 30/90 days | Project search latency at scale |

## Deployment requirement

Cluster-level access required. Platinum subscription required for anomaly detection. Coordinate with esdiag repo owner before deploying.
