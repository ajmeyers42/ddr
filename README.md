# DDR — Diagnostic-Driven Recommender

An AI agent and supporting Elastic cluster architecture for analyzing Elastic Stack diagnostic bundles and delivering customer-facing advisory recommendations.

## What This Is

The Diagnostic-Driven Recommender (DDR) is built on top of the [esdiag](https://github.com/elastic/esdiag) cluster — an existing Elastic deployment purpose-built for ingesting and exploring diagnostic bundles. DDR adds an advisory layer: rather than diagnosing problems reactively, it surfaces proactive recommendations that help customers get more value from their existing Elastic capabilities, replace custom-coded solutions with native features, and plan for capacity and growth.

DDR is deployed into a **separate Kibana space** within the hosted esdiag cluster. It does not modify esdiag's core assets; it builds on top of them.

## Goals

1. **Build trust** by surfacing real problems and improvements grounded in diagnostic evidence
2. **Drive capability adoption** by mapping findings to Elastic features the customer already has access to — prioritizing built-in capabilities over custom-coded workarounds
3. **Enable informed planning** by producing 12-month capacity and cost projections from current ingest and usage metrics
4. **Support the account team** by producing findings the sales team can interpret without the agent needing to frame commercial implications explicitly

## Repository Structure

```
ddr/
├── agent/
│   └── system-prompt.md              # The DDR agent system prompt (deploy to Kibana Agent Builder)
│
├── architecture/
│   ├── tool-inventory-and-cluster-features.md   # Full tool spec + in-cluster feature design
│   └── esdiag-repo-gap-analysis.md              # What to leverage, change, clone from esdiag
│
├── elasticsearch/
│   ├── index-templates/              # New index templates for recommender indices
│   ├── ingest-pipelines/             # Pipelines for agent output (findings, projections)
│   ├── transforms/                   # Pre-aggregation transforms against esdiag data
│   └── ml-jobs/                      # Anomaly detection, forecasting, ELSER configs
│
├── kibana/
│   ├── dashboards/                   # Advisory dashboards (capacity, adoption, findings)
│   ├── data-views/                   # Data views for new recommender indices
│   └── alerts/                       # Space-scoped alerting rules
│
├── docs/
│   └── deployment-guide.md           # How to deploy DDR into the esdiag cluster
│
└── assets/                           # Supporting assets (diagrams, reference data)
```

## Relationship to esdiag

DDR does **not** fork or replace esdiag. It deploys into the esdiag cluster and queries the data streams esdiag creates. The `esdiag process` and `esdiag serve` commands remain the ingest mechanism — DDR reads from the indices they populate.

See [`architecture/esdiag-repo-gap-analysis.md`](architecture/esdiag-repo-gap-analysis.md) for a full breakdown of what is leveraged as-is, what changes require owner approval, and what is built net-new in the DDR space.

## Deployment Phases

**Phase 1 — Space-only (no esdiag owner approval needed)**
- Kibana space, data views, advisory dashboards, alerting rules
- Agent system prompt deployed to Kibana Agent Builder
- Agent queries existing esdiag indices via ES|QL

**Phase 2 — Cluster additions (propose to esdiag owner)**
- Transforms (pre-aggregated summaries for faster agent tools)
- ML anomaly detection and forecasting jobs
- ELSER semantic index for findings history
- New index templates for recommender-specific indices

**Phase 3 — Contributions back to esdiag**
- Capacity planning and feature adoption dashboards as upstream contributions
- Account metadata propagation fixes (if audit confirms the gap)

## Agent Modes

The DDR agent operates in three modes, routed by request:

| Mode | Trigger | Output |
|------|---------|--------|
| **Health & Improvement** | "what issues", "best practices", "key findings" | Severity-ranked findings with evidence and recommended actions |
| **Capability Adoption** | "adopt features", "enterprise features", "upgrade readiness", "migrate to" | Capability table mapped to findings; migration readiness assessment |
| **Capacity & Cost Projection** | "capacity estimate", "12-month projection", "budget" | Scenario-based storage/compute/cost projections |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

This repo is intended for collaboration between AJ Meyers and colleagues working on the DDR project. PRs and issues welcome.

## Related

- [elastic/esdiag](https://github.com/elastic/esdiag) — the underlying diagnostic cluster this deploys into
- [elastic/agent-skills](https://github.com/elastic/agent-skills) — Elastic's official agent skill library
- [elastic/support-diagnostics](https://github.com/elastic/support-diagnostics) — the diagnostic bundle generator
