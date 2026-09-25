# Architecture Decision Records

One decision per file. Append-only: reversals get a new ADR that supersedes the old one.

| # | Decision | Status | Date |
|---|---|---|---|
| [0001](0001-orchard-core-boundary.md) | Orchard Core is the host, not the domain model | Accepted | 2026-08-06 |
| [0002](0002-tenancy-and-market-scope.md) | Tenancy and market scope | **Proposed** | 2026-08-06 |
| [0008](0008-products-plans-and-entitlement.md) | Products, plans, and entitlement — Payer/Beneficiary, holder + consumption scope | **Proposed** | 2026-08-06 |
| [0013](0013-opening-identity-and-canonical-urls.md) | Job Opening identity and canonical URLs | Accepted | 2026-08-06 |

## Queued — not yet written

These are known decisions that must be made before or during Phase 1. Listed here so they don't get lost.

| # | Decision | Needed by |
|---|---|---|
| 0003 | Party/Person model and its relationship to Orchard identity | Phase 1 |
| 0004 | Organization definition, hierarchy, and effective-dated relationships | Phase 1 |
| 0005 | External identifier crosswalk and identity resolution strategy | Phase 1 |
| 0006 | Provenance, confidence, and temporality pattern for ingested/enriched data | Phase 1 |
| 0007 | Merge and split semantics for Organizations | Phase 1 |
| 0009 | Authorization model (roles, org-scoped permissions, delegation) | Phase 2 |
| 0010 | Cutover URL continuity. Inherits the canonical scheme from [ADR-0013](0013-opening-identity-and-canonical-urls.md); the principal work item is **legacy repost-cluster consolidation** if the old system minted a URL per repost — unconfirmed, and the first thing to establish | Before cutover |
| 0014 | Job posting compliance guardrails — NY §194-B wage range validation at publish time, snapshot retention | With the publishing flow |
| 0011 | AI guardrails: enrichment review workflow **and AEDT compliance** (NYC Local Law 144 — bias audit, candidate notice, ranking separable from search). See [current-state.md §10](../current-state.md#10-new-risk-automated-employment-decision-tools) | **Before any candidate scoring or ranking is designed** |
| 0012 | Price schedules, quantity bands, and contextual pricing | Phase 1 |

## Template

```markdown
# ADR-NNNN: <short imperative title>

- **Status:** Proposed | Accepted | Rejected | Superseded by ADR-NNNN
- **Date:** YYYY-MM-DD
- **Deciders:**
- **Related:**

## Context
What forces are at play? What makes this a real decision rather than an obvious one?

## Options
Each with For / Against / Verdict.

## Decision
What we chose, stated plainly.

## Consequences
Positive, negative, and what we're accepting. Include enforcement mechanism if any.

## Open questions
```
