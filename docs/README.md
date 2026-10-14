# CSJ Platform Documentation

Canonical source of truth for the CSJ platform rebuild. If a decision is not written down here, it is not a decision.

## Structure

| Path | Purpose | Change frequency |
|---|---|---|
| `blueprint.md` | Stable architecture: vision, language, capabilities, domain model, context boundaries | Rare — quarterly at most |
| `commercial-model.md` | Revenue shapes, pricing, entitlement mechanics, derived from source strategy documents | When the business model changes |
| `current-state.md` | What the live system actually does — entitlement mechanics, price schedule, URL inventory, data quality | When the live site changes |
| `adr/NNNN-*.md` | One decision per file: context, options, tradeoffs, consequences | Often — append, don't rewrite |
| `adr/README.md` | ADR index and status table | With each new ADR |
| `specs/*.md` | Implementation work items with runnable acceptance criteria | Superseded when the work ships |
| `business-review.md` | Business-level review of the plans: commercial and roadmap pushback, proposals not decisions | When plans change |
| `sources/` | Frozen, dated snapshots of external evidence cited by analysis documents; see `sources/README.md` | Append only — new exports, never edits |

**Specs are not decisions.** A spec turns already-decided architecture into buildable work; if writing one surfaces a genuine choice, that belongs in an ADR first. Specs reference `docs/` rather than restating it, and their acceptance criteria are commands to run, not properties to inspect.

| Spec | Status |
|---|---|
| [phase-0-foundations.md](specs/phase-0-foundations.md) | Ready to implement |

**Analysis documents** (`commercial-model.md`, `current-state.md`, and future siblings) sit between the blueprint and the ADRs: they record what the evidence says, distinguish findings from inferences, and feed decisions into ADRs. Items marked **[CONFIRM]** are inferences awaiting verification and must not be implemented against until confirmed.

**Source precedence.** Where sources disagree, the order is: (1) the **live system** and internal price sheets — what actually happens; (2) **strategy documents** — what was intended; (3) **inference**. Corrections are recorded in place with a dated banner and the superseded reasoning retained, never deleted — same rule as the ADRs, and for the same reason. Two corrections of this kind are already on the record ([current-state.md §1](current-state.md), [§3.2](current-state.md)), both caused by reading pricing copy instead of mechanism copy.

## Rules

1. **The blueprint holds what should not change.** Technology choices, security mechanisms, integration specifics, and AI approaches belong in ADRs, not the blueprint. The blueprint may *reference* an ADR; it must not restate it.
2. **ADRs are append-only.** A decision that is reversed gets a new ADR that supersedes the old one. The old one stays, marked `Superseded by ADR-NNNN`. We preserve history in our documentation for the same reason we preserve it in our data.
3. **Conflicts are surfaced, not silently resolved.** If new work contradicts an existing document, name the conflict explicitly before proposing a change.
4. **Conversational decisions don't count.** Anything agreed in discussion must land in a document before implementation begins.

## ADR statuses

- `Proposed` — written, awaiting decision
- `Accepted` — decided; implementation may proceed
- `Superseded by ADR-NNNN` — replaced
- `Rejected` — considered and declined (kept, because the reasoning has value)

## Implementation tooling

Implementation is done with **Claude Code** working directly in the repository. Three consequences for how these documents are written and used:

1. **`/CLAUDE.md` is the enforcement surface.** It loads into every session automatically, so architectural invariants live there in condensed form. `docs/` holds the reasoning; `CLAUDE.md` holds the rules that must never be violated. When an ADR changes an invariant, update `CLAUDE.md` in the same commit — otherwise the reasoning and the rule drift apart.
2. **Specifications are referential, not self-contained.** An agent working in the repo can read `docs/`. Task specs should point at the canonical section rather than restating it; restated context is how documents go stale.
3. **Guardrails are executable wherever possible.** A convention that is only documented will eventually be violated by plausible-looking code that works. Prefer a failing build or test. The first and most important is enforcing that `CSJ.Domain.*` never references `OrchardCore.*` — an MSBuild target plus assembly-closure tests in `CSJ.ArchitectureTests` — because [ADR-0001](adr/0001-orchard-core-boundary.md)'s boundary is otherwise invisible in code. Development is on Windows; guardrails are C# and MSBuild, never shell scripts.

## External references

| Source | Use |
|---|---|
| [OrchardCMS/OrchardCore](https://github.com/OrchardCMS/OrchardCore) | **Authoritative for version and release state.** Stable is **v3.0** (`release/3.0`); meta-package `OrchardCore.Application.Cms.Targets`. |
| [docs.orchardcore.net](https://docs.orchardcore.net/en/latest/) | Primary Orchard Core documentation — behaviour, concepts, module reference. |
| [OrchardCMS/OrchardCore issues](https://github.com/OrchardCMS/OrchardCore/issues) | Known limitations and workarounds. Cite issue numbers; re-check status before relying on one. |

> ⚠️ **The docs site lags the release.** As of 2026-08-06 the documentation homepage stated the latest version was 2.1.5 while the stable release was 3.0. **Take version and release facts from the repository, behaviour from the docs** — and don't infer that a documented behaviour is current just because the page renders under `/en/latest/`.

Claims about framework behaviour should cite primary documentation. Where an ADR rests on training-data recall rather than a verified source, say so in the ADR.

## Naming

Use **CSJ**, never "CharterSchoolJobs", in namespaces, project names, schema names, and documentation.
