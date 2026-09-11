# Data Views

Data views for the DDR Kibana space. These are created within the DDR space and point to both existing esdiag indices and new recommender indices.

**Status**: Definitions pending — need actual esdiag field names first.

## Planned data views

| Data View Name | Index Pattern | Phase |
|---------------|--------------|-------|
| `DDR — Diagnostics` | `esdiag-*` (or actual esdiag pattern) | 1 |
| `DDR — Findings` | `recommender-findings-*` | 2 |
| `DDR — Projections` | `recommender-projections-*` | 2 |
| `DDR — ML Results` | `.ml-anomalies-recommender-*` | 2 |
| `DDR — Customer Context` | `recommender-customer-context-*` | 2 |

## First step

Confirm the actual index patterns deployed in the esdiag cluster:
```
GET _data_stream/esdiag-* 
```
