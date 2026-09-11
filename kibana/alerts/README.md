# Alerting Rules

Space-scoped alerting rules deployed within the DDR Kibana space.

**Status**: Not yet built. Phase 1 priority (basic rules), Phase 2 (ML-based rules).

## Planned rules

| Rule | Trigger | Action |
|------|---------|--------|
| Critical findings on new diagnostic | Agent writes one or more CRITICAL findings | Notify via connector |
| Storage forecast threshold | ML forecast projects full disk within 90 days | Trigger capacity review |
| ILM compliance degradation | Coverage drops >10 points since last diagnostic | Flag as regression |
| Follow-up overdue | A flagged finding has passed its due date | Notify owner |

## Notes

- Rules are space-scoped and do not affect esdiag's default space
- ML-based rules (storage forecast threshold) require Phase 2 ML jobs to be running
