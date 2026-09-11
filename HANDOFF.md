# DDR Project Brief
### Diagnostic-Driven Recommender — Handoff Document for Agents and Collaborators

**Repo**: https://github.com/ajmeyers42/ddr  
**Project**: DDR — Diagnostic-Driven Recommender (Claude Project)  
**Status**: Architecture complete, Phase 1 build ready to begin  
**Last updated**: September 2026

---

## What This Is

DDR is an AI agent and supporting Elastic cluster architecture that analyzes Elastic Stack diagnostic bundles and produces advisory recommendations for customers. It is built on top of the [esdiag](https://github.com/elastic/esdiag) cluster — an existing hosted Elastic deployment that ingests, parses, and indexes diagnostic bundles from customer clusters.

The agent is designed for use by Elastic Solution Architects and Field Engineers during customer engagements. It produces output written directly for the customer — not for Elastic's internal commercial purposes. A sales team reading the output should be able to infer the commercial implications from the facts and findings presented, without the agent making those implications explicit.

---

## The Problem Being Solved

Field teams currently run diagnostic analysis manually and ad-hoc. When a customer shares a diagnostic bundle, the SA has to manually review dozens of API output files, identify issues, cross-reference them with known best practices, and produce a written summary — all without a structured framework, repeatable format, or institutional memory of prior similar analyses.

DDR automates and structures that process, producing consistent, evidence-grounded findings across three scenarios a customer might request:

1. **Health & improvement review** — what's wrong, what can be done better
2. **Capability adoption assessment** — what features they have access to but aren't using, and what it takes to adopt them; upgrade and migration readiness
3. **Capacity and cost projection** — 12-month forward estimates for storage, compute, and cost based on current metrics

---

## Key Design Decisions

These decisions were made deliberately and should not be re-litigated without a clear reason:

**Output is always customer-facing.** The agent writes for the customer's team. It does not frame findings in terms of Elastic's revenue, license tiers to sell, or consumption growth. It frames findings in terms of what the customer can do better, what they already have access to and aren't using, and what they need to plan for. Sales implications are visible to anyone reading it — they don't need to be stated.

**Evidence discipline is non-negotiable.** Every finding must cite the specific diagnostic artifact it comes from. If a metric isn't in the data, it isn't in the finding. Inferred findings are labeled `[Inferred]` with explicit reasoning. No fabricated values.

**Native capabilities over custom code.** The agent's core recommendation posture is: where Elastic already solves a problem with a built-in, supported feature, recommend that over whatever custom solution the customer has built. This applies to ILM over cron jobs, Elastic Agent over custom Beats configs, ingest pipelines over Logstash transforms that belong closer to the source, Watcher over external schedulers, and so on.

**DDR does not replace esdiag — it builds on top of it.** The ingest layer (parsing, enriching, and indexing diagnostic bundles) is fully handled by esdiag's Rust processor. DDR queries the data esdiag creates. DDR does not duplicate, fork, or replace any esdiag capability.

**Separate Kibana Space, additive cluster assets.** DDR deploys into its own Kibana space within the esdiag cluster. Kibana-layer assets (dashboards, data views, alerting rules) are space-isolated and can be deployed without esdiag owner approval. Cluster-level assets (transforms, ML jobs, index templates) are additive — they don't modify esdiag's existing assets — but do require coordination with the esdiag owner before deployment.

---

## Architecture Overview

### The Two Layers

```
┌─────────────────────────────────────────────────────────┐
│                   DDR LAYER (new)                       │
│                                                         │
│  Agent (system prompt)  ←→  Agent Tools (6 categories) │
│         ↓                           ↓                   │
│  Kibana Space: dashboards, data views, alerts           │
│  Elasticsearch: transforms, ML jobs, findings indices   │
└──────────────────────┬──────────────────────────────────┘
                       │ reads from
┌──────────────────────▼──────────────────────────────────┐
│                  ESDIAG LAYER (existing)                 │
│                                                         │
│  esdiag process/serve → indexed diagnostic data streams │
│  esdiag setup → index templates, ingest pipelines,      │
│                  Kibana dashboards, data views           │
│  .agents/skills/esdiag/ → AI agent skill for esdiag CLI │
└─────────────────────────────────────────────────────────┘
```

### Agent Modes

The agent routes to one or more modes based on request phrasing:

| Mode | Triggers | Output |
|------|----------|--------|
| **1 — Health & Improvement** | "what issues", "best practices", "what can be done better", "key findings" | Severity-ranked findings (CRITICAL / HIGH / MEDIUM / OPPORTUNITY) with evidence citations and recommended actions |
| **2 — Capability Adoption** | "adopt features", "enterprise features", "upgrade readiness", "migrate to X" | Two-table output: capabilities included in current subscription not yet in use; capabilities available at higher subscription level — both mapped to real diagnostic findings |
| **3 — Capacity & Cost Projection** | "capacity estimate", "12-month projection", "budget", "what will this cost" | Conservative / base / aggressive scenario table for storage, compute, and cost; cost reduction levers available today |

### Agent Tool Categories (6)

1. **Diagnostic Retrieval** — list, get, search, compare diagnostics; get customer context
2. **Cluster Metrics & State** — query pre-aggregated transform outputs (health, nodes, indices, ILM, shards, slow queries, ingest pipelines)
3. **ML & Anomaly** — get anomaly results, forecasts, find similar clusters via ELSER, search findings history
4. **Knowledge & Docs** — search Elastic docs (ELSER, version-scoped), get version compatibility matrix, feature availability
5. **Visualization Access** — list/link dashboards, get data views, run ad-hoc ES|QL
6. **Output & Persistence** — write findings, write projections, create summary reports, flag for follow-up

Full tool spec: `architecture/tool-inventory-and-cluster-features.md`

### In-Cluster Features Supporting the Tools

| Layer | What It Provides |
|-------|-----------------|
| **Ingest** (esdiag — existing) | Diagnostic parsing, enrichment, account/case/opportunity tagging |
| **Transforms** (DDR — new, Phase 2) | Pre-aggregated summaries: cluster health, node metrics, index growth, ILM compliance, shard distribution, slow query shapes, custom code inventory |
| **ML** (DDR — new, Phase 2) | 7 anomaly detection jobs, 3 forecasting jobs, ELSER semantic index for findings history |
| **Kibana** (DDR — new, Phase 1) | Advisory dashboards, data views, space-scoped alerting rules |

---

## Repository Structure

```
ddr/
├── agent/
│   └── system-prompt.md              ← Deploy this to Kibana Agent Builder
│
├── architecture/
│   ├── tool-inventory-and-cluster-features.md   ← Full tool spec + in-cluster feature design
│   └── esdiag-repo-gap-analysis.md              ← What exists in esdiag, what DDR adds
│
├── elasticsearch/
│   ├── index-templates/     ← Templates for recommender-* indices (Phase 2, build next)
│   ├── ingest-pipelines/    ← Pipelines for findings/projections output (Phase 2)
│   ├── transforms/          ← Pre-aggregation transforms (Phase 2, build next)
│   └── ml-jobs/             ← Anomaly detection + forecasting (Phase 2)
│
├── kibana/
│   ├── dashboards/          ← Advisory dashboards (Phase 1, build next)
│   ├── data-views/          ← DDR data views (Phase 1, build next)
│   └── alerts/              ← Space-scoped alerting rules (Phase 1)
│
└── docs/
    └── deployment-guide.md  ← Step-by-step deployment instructions
```

---

## External Dependencies and Access Points

| Resource | URL / Location | Notes |
|----------|---------------|-------|
| DDR repo | https://github.com/ajmeyers42/ddr | Public, main branch |
| esdiag repo | https://github.com/elastic/esdiag | Read reference; do not fork or modify without owner approval |
| esdiag agent skill | `elastic/esdiag/.agents/skills/esdiag/` | Include in agent context for collect/process workflows |
| esdiag LLM setup guide | `elastic/esdiag/docs/` | Follow for LLM connector configuration |
| esdiag v0.16.4 | https://docs.rs/crate/esdiag/latest | Current stable release as of project start |
| Elastic support diagnostics | https://github.com/elastic/support-diagnostics | The tool that generates the diagnostic bundles |

---

## What Is Not Decided Yet

These are open questions that need answers before certain build tasks can proceed:

1. **Actual esdiag field names** — all transforms, ML jobs, and ES|QL queries must be written against real field names from the deployed index templates. Run `GET _index_template/esdiag-*` against the hosted esdiag cluster to get them. This is the first task for any agent beginning Phase 2 work.

2. **esdiag cluster access** — the hosted esdiag cluster endpoint and API key needed to deploy DDR assets and run the agent. The esdiag LLM setup guide covers the connection pattern.

3. **esdiag owner coordination** — Phase 2 cluster-level assets need owner approval. The recommended path is opening issues or PRs on `elastic/esdiag` proposing: (a) account metadata propagation fix, (b) `setup --profile recommender` command for DDR's cluster assets.

4. **MCP endpoint** — a new MCP server exposing DDR tools for external agent access (Claude, Cursor, or other agentic frameworks) is intended but not yet designed. This is Phase 3 scope.

---

*Document version: 1.0*  
*Repo: https://github.com/ajmeyers42/ddr*
