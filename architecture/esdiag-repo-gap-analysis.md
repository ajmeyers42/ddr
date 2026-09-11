# Diagnostic-Driven Recommender Agent
## esdiag Repo Gap Analysis — What to Leverage, Change, Clone, and Build

---

## What esdiag Actually Is (And What That Changes)

Before mapping anything, it's important to understand what the repo has actually built. esdiag is not just a diagnostic loading tool — it is a production-grade Rust application that handles the full ingest lifecycle:

- **Parses and enriches** raw diagnostic bundle archives (zip/directory) into structured Elasticsearch documents via its own internal processor pipeline (`src/processor/`)
- **Deploys all cluster assets** via `esdiag setup`: index templates, ingest pipelines, Kibana dashboards, data views, and saved searches — everything the cluster needs to operate
- **Provides a web UI and desktop app** for uploading and processing bundles
- **Collects directly from live clusters** via `esdiag collect`
- **Tags diagnostics with account metadata** at ingest time: `--account`, `--case`, `--opportunity`, `--user` flags on `esdiag process`
- **Has an AI agent skill** at `.agents/skills/esdiag/` — a pre-built skill that teaches AI agents how to use esdiag's CLI and workflow
- **Has an LLM setup guide** (added 0.16.0) — documenting how to configure AI assistant resource connections against the cluster
- **Supports the full deployment spectrum**: ECH, self-managed, ECK, ECE, Logstash, Kibana collection

The key implication: **the ingest layer I designed from scratch in the previous architecture document already exists in esdiag.** The field names, index patterns, mapping structure, and enrichment logic are defined inside the Rust processor code and the assets deployed by `esdiag setup`. The recommender agent is built *on top of* this, not alongside it.

---

## Deployment Constraint: Separate Kibana Space

A Kibana Space provides isolation for **Kibana-layer assets only**:
- Dashboards, visualizations, saved searches, data views, canvas workpads — all space-specific ✓
- Alerting rules — space-specific ✓

The following are **cluster-level and space-agnostic** — they exist once per cluster regardless of how many spaces are configured:
- Index templates and component templates
- Ingest pipelines
- Transforms (pivot and latest)
- ML jobs (anomaly detection, forecasting, trained models)
- The actual indices and data streams

This split is the central fact governing the deployment plan. Everything the recommender agent needs at the Kibana layer can be deployed independently into the new space. Everything at the cluster layer requires coordination with the esdiag owner.

---

## Asset Inventory: What the Repo Has

Based on the README, CHANGELOG history, and AGENTS.md, esdiag's `assets/` directory (deployed via `esdiag setup`) contains:

| Asset Category | What Exists | Evidence |
|----------------|-------------|----------|
| **Index templates** | Data stream templates for all diagnostic document types: cluster state, node stats, index stats, shard data, ILM data, mappings summaries, snapshot data, task exports, Logstash data, Kibana data | 0.11–0.16 CHANGELOG; `esdiag setup` installs these; `src/setup.rs` drives them |
| **Ingest pipelines** | Enrichment pipelines for cluster metadata, node identity lookup, shard enrichment, failure store enrichment, data stream metadata tagging | 0.11, 0.14 CHANGELOG entries: "shard statistics enrichment", "failure store enrichment", "node-derived metrics" |
| **Kibana dashboards** | Diagnostic analysis dashboards (support/troubleshooting focus); cluster report dashboard with pre-filtered Kibana link | 0.13 "Added Kibana assets to setup command"; Kibana link output in README |
| **Data views** | Data views for all esdiag data streams | 0.13 "Kibana setup support"; "Updated dashboard IDs to human-readable values" |
| **Saved searches** | Saved searches for diagnostic exploration | Kibana assets confirmed in 0.13 |
| **Agent skill** | `.agents/skills/esdiag/` — AI agent skill for using esdiag CLI | AGENTS.md key paths |
| **LLM setup guide** | Documentation for connecting AI assistants (Kibana AI assistant or external) to the esdiag cluster | 0.16.0: "Added a comprehensive LLM setup guide" |

**What the ingest processor already does** (inside `src/processor/`, not as standalone pipelines, but as the Rust processing layer that produces the indexed documents):
- Parses raw API JSON output from diagnostic bundles into typed document streams
- Enriches with cluster metadata, node identity, diagnostic ID and version (`diagnostic.id`, `diagnostic.version`)
- Tags with account metadata passed via `--account`, `--case`, `--opportunity`, `--user`
- Handles ECK/Kubernetes platform metadata (0.16.0)
- Handles Logstash sub-streams with their own templates (0.16.0)
- Produces mapping summaries as part of index statistics (0.14)

---

## Gap Analysis: Three Categories

### A — Leverage As-Is (No Changes Needed)

These esdiag assets directly serve the recommender agent without modification. The agent tools should query against them exactly as deployed.

**`esdiag process` / `esdiag serve` — the ingest mechanism**
The recommender agent does not need to re-implement diagnostic loading. When a new diagnostic needs to be analyzed, the workflow is: upload via the esdiag web UI or CLI → processed into the existing data streams → agent queries from there. The `--account`, `--case`, `--opportunity` flags already provide the customer tagging the agent needs. The agent should surface these flags in its session opening so the account team knows to use them.

**Existing index templates and data streams**
The field names and structure coming out of esdiag's processor are the source of truth. The agent's ES|QL queries, transforms, and ML jobs must be written against these actual field names — not the hypothetical ones from the earlier architecture document. Before building transforms, the first task is reading the actual field mappings from the deployed templates.

**Existing Kibana dashboards and data views**
The support/troubleshooting dashboards already exist and are useful context. The recommender agent's `get_dashboard_url` tool can link to these existing dashboards. The new space gets its own dashboards, but the existing ones don't need to be replaced — the agent can reference both.

**`.agents/skills/esdiag/` — the esdiag agent skill**
This is a ready-made skill that teaches an AI agent how to use esdiag's CLI commands (`collect`, `process`, `setup`, `serve`, `host`, `keystore`). The recommender agent should reference and incorporate this skill — it defines how to collect a new diagnostic from a customer cluster, which is a core workflow that feeds the recommender agent.

**`example.env` and the `ESDIAG_OUTPUT_*` variable pattern**
The environment variable configuration pattern is stable and usable as-is for connecting agent tools to the cluster.

**The account metadata fields on `esdiag process`**
`--account`, `--case`, `--opportunity`, `--user` — these are the customer tagging mechanism. The recommender agent's `list_diagnostics` and `search_diagnostics` tools should filter on these fields. No changes needed; the metadata is already there if bundles were loaded with these flags.

---

### B — Recommend Changing (Requires Owner Approval)

These are gaps or limitations in the existing esdiag assets that affect the recommender agent's usefulness. All changes should be proposed as PRs or issues to the esdiag repo owner, not implemented unilaterally.

**1. Ingest pipeline enrichment — customer context propagation**
The existing ingest pipelines enrich for cluster metadata and node identity, but it's unclear whether `account`, `case`, `opportunity`, and `user` (which are set at process-time in the Rust layer) propagate consistently through all sub-streams (Logstash, Kibana, ECK metadata streams). If sub-streams lose these fields, the recommender agent cannot filter by customer across all data types.

*Recommended change*: Audit field propagation across all sub-stream templates and add a shared component template that enforces `diagnostic.account`, `diagnostic.case`, `diagnostic.opportunity`, `diagnostic.user` mapping on all data streams. Propose as a PR.

**2. Kibana dashboards — advisory views missing**
The existing dashboards are built for support engineers diagnosing problems. They lack the forward-looking views the recommender agent needs: capacity trend + growth projection, ILM compliance coverage over time, feature adoption gap (what the cluster has vs. what it uses), custom code inventory.

*Recommended change*: Propose adding a "Capacity & Planning" dashboard and a "Feature Adoption Overview" dashboard to the main esdiag Kibana asset set. These are generally useful beyond the recommender agent use case — good candidates for a contribution.

**3. The `esdiag process` account metadata flags — documentation and convention**
The `--account`, `--case`, `--opportunity` flags exist but their consistent use is voluntary. If bundles get loaded without them, the recommender agent loses customer attribution.

*Recommended change*: Propose a convention document or a soft-validation warning when these fields are absent — e.g., a warning in the web UI upload flow when account name is blank. This protects the recommender agent's filtering capability without breaking anything.

**4. Cluster-level assets needed for the recommender agent**
Transforms and ML jobs are cluster-level assets. If the recommender agent needs them (it does), they cannot be deployed in a separate space — they need to go into the shared cluster. The owner's approval is required.

*Recommended change*: Propose a new `setup --profile recommender` option or a separate `esdiag setup-recommender` command that installs the additional transforms and ML jobs needed by the recommender agent. This keeps them managed within the esdiag repo's setup lifecycle rather than deployed ad-hoc.

---

### C — Clone/Modify or Build New in the Separate Space

These can be deployed independently into the new Kibana space without touching esdiag's assets or requiring owner approval — with one important caveat on cluster-level assets noted below.

**Kibana layer (space-isolated — deploy freely):**

| Asset | Approach |
|-------|----------|
| New data views | Create in the new space pointing to existing esdiag indices plus new recommender indices. Name them `Recommender — [Category]` to distinguish from esdiag's originals. |
| Advisory dashboards | Build new: Capacity & Planning, Feature Adoption Assessment, Findings History, Multi-Cluster Comparison, ML Anomaly Summary. These complement rather than replace esdiag's existing dashboards. |
| Saved searches | New saved searches scoped to advisory and capacity use cases. No conflict. |
| Alerting rules | Space-specific alerting: critical findings trigger, storage forecast threshold, ILM compliance degradation. Deploy in the new space. |

**Cluster layer (require coordination but don't modify esdiag assets):**

These are net-new cluster assets that don't conflict with esdiag's existing setup. They should be proposed as additions — either via a contribution to the repo or via a separate deployment process approved by the owner.

| Asset | Approach |
|-------|----------|
| Transforms | New pivot and latest transforms against existing esdiag indices. Output to new index patterns (`recommender-*`). These read esdiag data but don't modify it. |
| ML anomaly detection jobs | New jobs against existing esdiag data streams. Outputs to `.ml-anomalies-*`. No conflict. |
| ML forecasting jobs | New forecasting jobs. No conflict. |
| ELSER semantic index | New index `recommender-findings-*` with ELSER encoding. Fully additive. |
| New index templates | Only for net-new recommender indices: `recommender-findings-*`, `recommender-projections-*`, `recommender-customer-context-*`. These don't touch esdiag's existing templates. |
| New ingest pipelines | Only for agent output (findings, projections). These are for new indices, not modifications to esdiag's pipelines. |

**The agent system prompt and skill files — build new, no conflicts:**
The recommender agent's system prompt, tool definitions, and any agent skill files are entirely new and don't conflict with anything in the esdiag repo.

---

## Revised Architecture — What Changes From the Previous Design

The previous architecture document assumed building the ingest layer from scratch. Now that we know esdiag already handles it, here is what changes:

### What Goes Away
- `diag-bundle-parser` pipeline → **replaced by esdiag's processor** (Rust-based, already deployed)
- `diag-enrichment` pipeline → **largely replaced** by esdiag's existing enrichment; only the customer context propagation gap (Category B above) needs addressing
- `diag-slow-query-normalizer` → **check if esdiag already parses slow logs**; if so, this is already done

### What Stays But Needs Field Name Mapping
Every transform query, ML job, ES|QL tool query, and data view definition in the previous architecture needs to be rewritten against esdiag's actual field names. The canonical source is the index template mappings deployed by `esdiag setup`. Before any transform or ML job is written, the field map needs to be read from:

```esql
GET _index_template/esdiag-*
```

or from the `gen/schemas` directory in the repo.

### What's Fully Additive and Unchanged
The transform layer, ML layer, Kibana layer, and agent tool definitions remain as designed — they are additive to the esdiag cluster and don't conflict. They just need to reference real field names instead of hypothetical ones.

### What the Agent Prompt Should Add
The recommender agent system prompt should be updated to include:

1. **How to use esdiag** — reference the `.agents/skills/esdiag/` skill for collecting diagnostics from customer clusters and loading them into the esdiag cluster
2. **Space context** — the agent operates in a named Kibana space; links to dashboards and data views must be space-qualified
3. **Account metadata requirement** — when interpreting a diagnostic, check whether `diagnostic.account` / `diagnostic.case` / `diagnostic.opportunity` are populated; if not, note that the bundle was loaded without customer attribution and flag it

---

## Deployment Sequence

Given the space isolation constraint and the need for owner coordination, the deployment naturally sequences into three phases:

**Phase 1 — Space-only deployment (no owner approval needed)**
- Create new Kibana space for the recommender agent
- Create data views pointing to existing esdiag indices
- Deploy advisory dashboards (capacity, adoption, findings history)
- Deploy space-specific alerting rules
- Configure recommender agent with system prompt, tools referencing existing esdiag data

The agent is functional at this point with ES|QL-based ad-hoc analysis against the existing data. No transforms or ML yet — the agent compensates with direct queries and on-session aggregation.

**Phase 2 — Cluster additions (propose to owner, deploy once approved)**
- Submit PR or proposal: new component template for customer metadata field propagation
- Submit PR or proposal: `setup --profile recommender` for transforms and ML jobs
- Once approved: deploy transforms (pre-aggregated summaries, faster agent tools)
- Once approved: deploy ML jobs (anomaly detection, forecasting)
- Once approved: deploy ELSER semantic indexing pipeline for findings history

The agent gains proactive anomaly surfacing, faster metric queries, semantic similarity search across prior findings.

**Phase 3 — Contribution back (where applicable)**
- Propose advisory dashboards as contributions to esdiag's Kibana asset set — capacity planning and feature adoption views are useful to any esdiag deployment
- Propose account metadata propagation fix as a PR if the audit confirms the gap
- Propose LLM/agent integration improvements informed by what the recommender agent learns from real usage

---

## Summary Table

| esdiag Asset | Action | Rationale |
|-------------|--------|-----------|
| `esdiag process` / `serve` / `collect` | **Leverage as-is** | This is the ingest mechanism; don't duplicate it |
| Index templates (all `esdiag-*`) | **Leverage as-is** | Field structure is source of truth; write all queries against it |
| Ingest pipelines (existing) | **Leverage as-is + propose gap fix** | Mostly covers the need; account metadata propagation across sub-streams needs audit |
| Kibana dashboards (existing) | **Leverage as-is + add new** | Reference existing in agent; add advisory dashboards in new space |
| Data views (existing) | **Leverage as-is + add new** | Add recommender-specific views to new space |
| `.agents/skills/esdiag/` | **Leverage as-is** | Include in recommender agent context for collect/process workflows |
| LLM setup guide | **Leverage as-is** | Follow its configuration patterns for connecting the agent |
| Transforms | **Build new (propose to owner)** | Net-new, cluster-level; need owner approval to deploy |
| ML jobs | **Build new (propose to owner)** | Net-new, cluster-level; need owner approval to deploy |
| ELSER semantic index | **Build new (propose to owner)** | Net-new, cluster-level; need owner approval to deploy |
| Recommender findings/projections index templates | **Build new (propose to owner)** | New indices only; don't overlap with esdiag templates |
| Agent output ingest pipelines | **Build new (propose to owner)** | For new indices only; no conflict with existing |
| Advisory dashboards | **Build new in space** | Space-isolated; deploy independently |
| Recommender data views | **Build new in space** | Space-isolated; deploy independently |
| Space alerting rules | **Build new in space** | Space-isolated; deploy independently |
| Agent system prompt | **Build new** | Entirely new; update to reference esdiag skill and space context |

---

*Document version: 1.0*
*References: elastic/esdiag v0.16.4 — https://github.com/elastic/esdiag*
