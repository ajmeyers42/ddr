# Contributing to DDR

## Overview

DDR is a living project — the agent prompt, architecture, and cluster assets will evolve as we learn from real diagnostic sessions. This document covers how to contribute effectively.

## What We're Working On

The current priorities, roughly in order:

1. **Validate the agent prompt** against real diagnostic sessions and refine based on output quality
2. **Map esdiag field names** — read the deployed index template mappings and update all queries in the tool inventory to use real field names
3. **Build Phase 1 assets** — Kibana data views, advisory dashboards, alerting rules for the DDR space
4. **Propose Phase 2 assets** to the esdiag owner — transforms, ML jobs, new index templates

## Repository Conventions

### Branches

- `main` — stable, reviewed content only
- `draft/*` — work in progress on agent prompt or architecture docs
- `feature/*` — new Elasticsearch or Kibana assets
- `fix/*` — corrections to existing content

### Commit messages

Use a short prefix: `agent:`, `arch:`, `es:`, `kibana:`, `docs:`

Examples:
```
agent: refine mode 2 capability adoption output format
es: add cluster-health-summary transform definition
kibana: add capacity trend dashboard skeleton
docs: update deployment guide with phase 1 steps
```

### Pull Requests

- Link to the relevant issue if one exists
- For agent prompt changes, include a before/after example showing the change in output
- For ES/Kibana asset changes, include the tested API request and response in the PR description

## Agent Prompt Changes

The system prompt is in `agent/system-prompt.md`. When proposing changes:

- Test the change against at least one real or sample diagnostic session
- Document what specific output behavior the change improves or corrects
- Keep the customer-value framing intact — recommendations should read as advisor output, not vendor pitch

## Elasticsearch Assets

Assets in `elasticsearch/` are JSON files intended to be deployed via the Elasticsearch API. Each file should include a comment header (in an accompanying `.md` file) explaining:

- What it does
- Which esdiag indices it reads from
- What new index it writes to (for transforms and pipelines)
- What owner approval is required before deployment

## Kibana Assets

Kibana assets in `kibana/` are exported saved object JSON (NDJSON format from Kibana's export API). Export with `Stack Management → Saved Objects → Export` with dependencies included.

Each dashboard, data view, or alert should have an accompanying `README.md` describing its purpose and which data view it depends on.

## Issues

Use GitHub Issues for:
- Tracking work items per deployment phase
- Logging questions about esdiag field names or behavior that need investigation
- Flagging proposed changes to esdiag (Category B from the gap analysis) that need owner input

Label convention:
- `phase-1` / `phase-2` / `phase-3`
- `agent-prompt` / `elasticsearch` / `kibana`
- `needs-esdiag-owner` — anything requiring approval before it can be deployed
- `field-mapping` — anything blocked on knowing the actual esdiag field names
