# Business Review — Commercial and Roadmap Pushback

**Status:** Analysis, v0.1 — proposals, not decisions
**Date:** 2026-08-07
**Sources:**

- [`blueprint.md`](blueprint.md) v0.1 (2026-08-06)
- [`commercial-model.md`](commercial-model.md) v0.1
- [`current-state.md`](current-state.md) v0.1
- [ADR-0002](adr/0002-tenancy-and-market-scope.md), [ADR-0008](adr/0008-products-plans-and-entitlement.md)

This document is a business-level review of the plans above. It deliberately does not re-litigate the architecture — the invariants in `CLAUDE.md` (Orchard boundary, single entitlement service, history preservation, ubiquitous language) are sound and none of the findings below touch them. The pattern in the findings is the opposite failure mode: the architectural reasoning is strong enough that it has been absorbing attention the *commercial* layer needed.

Per the documentation rules, each finding names its conflict or gap against a specific document before proposing a change. Consolidated proposed edits are in §9; new open questions in §10. **Nothing here is decided until Kent accepts the edits into the canonical documents.**

---

## 1. The supply side is unanalyzed — and it is the actual product

**Gap, not conflict. The largest finding in this review.**

Three documents dissect employer pricing to the cent. There is no comparable analysis of candidates anywhere in the corpus:

- No figure for current application volume, applications per publication, or their trend.
- No answer to *why a candidate applies on CSJ rather than Indeed, LinkedIn, or a district site* — the platform's supply-side value proposition is asserted (free tools, §2.1 of commercial-model) but never examined.
- No candidate-liquidity KPI anywhere in the KPI discussion, which is otherwise thorough.
- Candidate profiles are deferred to Phase 3, and the candidate-side capability table (blueprint §3.2) is five rows against the demand side's nine-plus-nine.

Yet the Phase 1 exit criterion — *"a real employer buys a plan, posts a job, and **receives an application**"* — silently depends on candidate liquidity. Every employer product in the catalog is, ultimately, access to the candidate audience; the *"most capable and academically accomplished"* mission claim is a supply-side claim; and the standing constraint that candidates never pay is justified by a supply-advantage argument that has never been measured.

**What this blocks if unaddressed:** if applications-per-publication is weak or declining, no amount of entitlement modeling matters — the correct project is candidate acquisition, and the rebuild's sequencing should change to serve it.

**Proposed actions:**

1. **Before Phase 1 scope is frozen,** pull from the legacy system: applications per publication (distribution, not just mean), application trend over 24 months, candidate source mix (organic / job alert / syndication / fair), and repeat-applicant rate. Cheap queries; they either validate the roadmap or redirect it.
2. Add a supply-side analysis document (`candidate-side.md`) as a Phase 0 deliverable, peer to `commercial-model.md`.
3. Add **applications per publication-month** as a first-class KPI alongside the commercial set — it is the supply-side twin of the posting-month and the earliest warning of marketplace failure.
4. Blueprint edit at §9 below.

---

## 2. Repricing precedes the rebuild

**Conflict — with the implicit sequencing across all three documents. Commercial, and urgent.**

[current-state.md §3.3](current-state.md#33-the-plan-vs-à-la-carte-economics--corrected) establishes that Platinum delivers ~96% savings against à la carte — roughly $90,000 of posting-month value for $4,000 — and prices a 60-slot organization identically to a 200-slot one. The documents treat this as input to the data model ("tiers are rows") and defer the pricing itself as "a commercial decision."

That deferral undersells the finding. **Repricing is almost certainly the highest-ROI action available to the business, it requires zero code, and it is executable on the legacy system at the next renewal cycle.** A Custom tier above Platinum, a repriced Platinum, or both, plausibly funds a meaningful fraction of the rebuild.

The sequencing risk is specific: the Conversion Strategy's lifecycle features (*"here's what a plan would have saved you"*) compute against the current schedule. Built before repricing, they **institutionalize the underpricing** — every customer shown a 90%+ savings figure is a customer who will resist the correction later. The same applies to encoding the current tier ladder into launch marketing.

**Proposed actions:**

1. Treat pricing review as a **Phase 0 business workstream** parallel to the ADRs: Custom tier decision (blueprint open question #5), Platinum repricing, and the 25+-quantity à la carte gap (current-state Q5a) — settled before any pricing surface is rebuilt.
2. Sequence rule: **no lifecycle-savings feature ships before the price schedule it would advertise is the intended one.**
3. This does not touch the model. Tiers-as-rows, plan versioning, and price snapshots (blueprint §3.4 d–e) are exactly what makes repricing cheap — build them as planned.

---

## 3. Roadmap sequencing contradicts its own evidence

**Conflict — with blueprint §9.**

Two problems:

**a) Phase 2 (internal intelligence) precedes Phase 3 (fairs and hiring depth), but the corpus's own evidence points the other way.** Fairs are a live revenue line; the largest pipeline deal (Success Academy) is a win-back re-engaged *through a fair*; and the Conversion Strategy's central recommendation is fair↔plan connective tissue. Meanwhile Phase 2's payoff — enrichment, dashboards, CRM-driven sales — is a cost center until its pipeline converts, and it serves the §4 dataset thesis, which is itself unproven (finding 4). A solo developer's year is the scarcest resource in the company; spending it on internal tooling while the revenue-facing surfaces wait needs stronger justification than "the Phase 1 investment pays off here."

**b) Sector expansion sits in Phase 5 despite ADR-0002 claiming a new market is "essentially one table, one column."** If the claim is true, deferring its proof four phases forfeits cheap revenue experiments (e.g., `publicschooljobs` as a filtered view over the same graph). If it is false — because each sector needs its own candidate liquidity, which finding 1 suggests is the real cost — the architecture claim is misleading and should be corrected. Either way, waiting until Phase 5 to find out is the wrong test schedule. *(This does not mean launching a sector before the core is stable; it means the first sector experiment should be a deliberate early probe, not a distant phase.)*

**Proposed revision (blueprint §9):**

- **Phase 2 → "Fairs + conversion tissue":** career fair lifecycle (booth purchase, the flat-20% benefit entitlement surfaced at quote time, registration), plus the minimal CRM needed to work the fair-to-plan funnel. This monetizes the Phase 1 entitlement work immediately and serves the win-back motion.
- **Phase 3 → "Hiring depth + intelligence":** full posting lifecycle, candidate profiles, application management, and the enrichment pipeline — intelligence lands *after* there is a sales motion to consume it.
- **Add to Phase 2 or 3, explicitly:** a **sector-market probe** — stand up one additional Market (per ADR-0002's mechanism, once its open question 2 is settled) with a bounded marketing spend, measured on candidate liquidity, to price the expansion thesis with data.

---

## 4. "The connected dataset is the product" is a thesis, not a fact

**Conflict — with the framing of blueprint §1.1, which states it as settled.**

The claim justifies the identifier crosswalk, the provenance chain, the Intelligence context, and their Phase 1–2 position — "foundational rather than decorative." But current revenue is postings and fairs; the revenue shapes the dataset uniquely enables (6 advertising, 7 metered verification, 8 sponsored services) are all speculative; and part of the supporting argument is derived from the 2022 mission statement, which is thin evidence for a data-platform strategy.

The thesis may well be right — the identity-resolution evidence in current-state §7 is real, and CertifiedK12's stated value depends on the unforked graph. The problem is *presentation as fact*, because it exempts the graph investments from the right-sizing test every other component must pass (blueprint §6: "every piece of infrastructure is a claim on that budget"). A future reader should see the graph spend as a **priced bet on shapes 6–8 and on sector expansion**, revisited if those shapes stay hypothetical.

**Proposed edit (blueprint §1.1):** reframe the paragraph to state the two-sided truth — *the dataset is the platform's compounding asset and the strategic bet; postings and fairs are the product customers currently buy.* Add a falsifier: if by end of Phase 3 no revenue shape beyond 1–5 has a committed customer, the Intelligence investment plan gets re-reviewed. §9 has draft text.

---

## 5. Phase 1 is not a "thin revenue slice"

**Conflict — with blueprint §9's own description of Phase 1, and tension with §6's right-sizing principle.**

As scoped, Phase 1 contains: Organizations, effective-dated hierarchy, external identifier crosswalk, merge/split, Party/Person, provenance foundation, employer self-service, plan catalog, the four-kind entitlement service, payments, publication, and application delivery — plus the strangler-fig routing needed for any of it to serve production traffic. For one developer that is a year or more with nothing shipped, which is precisely the failure mode §9's own principle ("each phase must ship something") warns against.

There is a broader version of the same tension: §6 declares the binding constraint to be one person's ability to hold the system in their head, and the corpus then specifies eight bounded contexts, eight schemas and DbContexts, effective-dating on most relationships, and four entitlement kinds. Each is individually justified; collectively it is the complexity budget being spent on the domain model rather than infrastructure. Better than microservices — but still spent, and worth auditing with the same rigor §6 applies to message buses.

**Proposed cuts and consolidations (each reversible):**

1. **Merge/split: design for it, don't build it.** Keep the schema shape that makes merge possible (stable IDs, merge-event table, no destructive updates — blueprint §4.1e requires only this); defer the merge *tooling and workflow* until the first real merge. The Phase 1 requirement becomes "merges are representable," not "merges are operable."
2. **Crosswalk: seed, don't tool.** Load NCES/state IDs as data (an import script) rather than building crosswalk-management UI. The table from day one, per §4.1c; the tooling when Phase 2/3 needs it.
3. **Context consolidation question:** at this scale, do CRM, Intelligence, and Engagements need to be three contexts, or one "Relationships" context with three schemas' worth of tables? Fewer DbContexts is directly cheaper for the person holding the system in their head. Worth deciding in ADR form before the module skeletons are cut.
4. **Application delivery at minimum depth** (blueprint open question #13 resolved toward the cheapest option — likely email forwarding) for the Phase 1 exit criterion, deepening in Phase 3.

---

## 6. Time-to-fill — the headline metric — may be unmeasurable

**Gap — against blueprint §4.3, which lists time-to-fill first among what the Opening/Publication split "unlocks."**

The split is *necessary* for time-to-fill but not *sufficient*: `Opening.Filled` requires the employer to say so, and nothing in the current or planned product compels them to. Today employers let publications lapse; lapse is ambiguous between filled, cancelled, filled elsewhere, and forgotten. A metric computed from voluntary, unprompted closure events will be sparse and biased, and it is the number the platform's pitch (*"cost-effectively staff my school"*) leans on.

**Proposed actions:**

1. Decide the **fill-capture mechanism** as a product feature before the metric is promised anywhere: candidates are (a) a "still hiring?" nudge loop at publication lapse and non-renewal, (b) closure prompts embedded in the ATS-depth workflow (which makes capability-flag postings the instrumented population), (c) inference from repost cessation, clearly labeled as estimated. Likely all three, with provenance on which method produced each fill date — §4.4 discipline applied to the platform's own analytics.
2. Until capture exists, internal and customer-facing copy should say **time-to-lapse / repost count**, which are honestly measurable, not time-to-fill.
3. New open question at §10.

---

## 7. Bump economics are zero-sum — caution on two conclusions

**Conflict — with [current-state.md §3.3](current-state.md#33-the-plan-vs-à-la-carte-economics--corrected) consequences 1 and 2 (the bump-SKU gap and the "$3,000 of included bumps" argument).**

Board position is a **positional good**. The $50 implied bump price is real arithmetic, but it prices a bump *at current bump volume*. Two consequences the analysis doesn't carry through:

- **An explicit bump SKU dilutes itself.** Every incremental purchased bump devalues all others and pushes genuinely fresh postings down. Sold aggressively, it converges on a board whose top is stale-but-paid — degrading the candidate experience the marketplace depends on and sitting badly with the mission's match-quality claim (blueprint §1.0). The same dynamic, slower, applies to large-slot plans recycling continuously against the à la carte long tail.
- **The "$3,000 notional bump value" sales argument double-counts.** If every plan holder receives 12 bumps per slot per year, bumps are the ambient condition of the board, not a $50-each windfall. It is a fine argument that plans are *underpriced relative to à la carte* (§3.3 already shows that); it is not safe as a customer-facing value claim.

**Proposed actions:**

1. **No bump/refresh SKU without a marketplace-health analysis first:** expected bump volume at price points, effect on median time-on-top for fresh postings, and candidate-side staleness measures. This is a half-day of modeling that prevents a revenue feature from quietly eroding the supply side.
2. Keep the mechanic and its pricing coherence in the model (Publication/`RankedAt` as designed — the architecture is right); treat the *productization* of bumps as an open commercial question, not an obvious gap to fill.
3. Note the interaction with finding 1: any bump decision changes what candidates see first, so it belongs to the supply-side analysis as much as the price list.

---

## 8. The full-utilization correction may have overcorrected *(minor)*

**Against [current-state.md §3.3](current-state.md#33-the-plan-vs-à-la-carte-economics--corrected)'s retraction of commercial-model §1.3.**

The retraction generalizes from the two Platinum customers ("routinely exceed 50 concurrent") to the claim that full slot utilization is "the normal case." Platinum buyers are the customers *selected for* high utilization; Bronze and Silver buyers plausibly run well under capacity, in which case the honest lifecycle-savings feature will show weaker numbers precisely for the conversion segment it targets. The original concern was wrong in mechanism (postings-per-year) but may have been right in direction for the lower tiers.

**Proposed action:** one query against legacy data — actual concurrent-occupancy distribution by tier over the trailing 12 months — before any savings-messaging feature is specified. Cheap, and it settles whether §1.3's ghost is really dead.

---

## 9. Consolidated proposed edits to `blueprint.md`

For acceptance individually; none is applied yet.

**§1.1, replace the final paragraph:**

> The architectural implication of holding both: **the connected dataset is the compounding asset, and postings and fairs are the product customers buy today.** The organization/contact/signal graph is a deliberate bet — it is what makes revenue shapes 6–8 and sector expansion possible, and design decisions that fragment it are expensive even when locally convenient. But it is a bet with a review date: if by the end of Phase 3 no revenue shape beyond 1–5 has a committed customer, the scale of Intelligence investment gets re-reviewed rather than compounded.

**§3.2, add a row and a note:**

> | Candidate liquidity measurement | Applications per publication-month, source mix, repeat-applicant rate — the supply-side twin of the posting-month. *Phase 0 baseline from legacy data; ongoing KPI thereafter.* |
>
> The supply side is the product the demand side pays for. Its analysis lives in `candidate-side.md` *(to be written — Phase 0)*, peer to `commercial-model.md`.

**§4.3, qualify the time-to-fill claim:**

> **What this unlocks, none of it currently measurable:** time-to-fill *(requires a fill-capture mechanism — see open question #16; until one exists, the honest metrics are time-to-lapse and repost count)*, honest posting-month accounting per role, fill rate and repost rate, and suppressing duplicate-looking reposts for candidates who already applied.

**§9, roadmap:** swap Phase 2 and Phase 3 content per finding 3 (Fairs + conversion tissue before Intelligence), and add the sector-market probe as an explicit, bounded experiment in Phase 2–3. Add to Phase 0: pricing review workstream (finding 2) and the legacy data pulls (findings 1, 8).

**§9 Phase 1, narrow the scope sentence:**

> …merge/split **representable in schema (tooling deferred to first real merge)**, external identifier crosswalk **seeded by import (management tooling deferred)**, Party/Person, and the provenance foundation…

**§6, add after the infrastructure list:**

> The same test applies to the domain model itself. Eight bounded contexts, four entitlement kinds, and effective-dating on most relationships are each justified — but they draw on the same comprehension budget as infrastructure, and consolidation questions (e.g., CRM + Intelligence + Engagements as one context) deserve the same scrutiny as "do we need a message bus."

---

## 10. New open questions for blueprint §10

Numbering continues from #14.

| # | Question | Blocks |
|---|---|---|
| 15 | **What is current candidate liquidity?** Applications per publication (distribution and trend), candidate source mix, repeat-applicant rate — from legacy data. | Phase 1 scoping; validates or redirects the roadmap. (§1) |
| 16 | **How is "filled" captured?** Nudge loop, ATS-workflow closure, inference, or a combination — a product decision, prerequisite to promising time-to-fill anywhere. | §4.3's headline metric; Phase 3 analytics. (§6) |
| 17 | **Is repricing (Custom tier + Platinum correction) settled before any pricing surface or lifecycle-savings feature is rebuilt?** Sequencing constraint, not just the open commercial question in #5. | Lifecycle marketing features; launch pricing pages. (§2) |
| 18 | **Should a bump/refresh SKU exist at all**, given board position is zero-sum? Requires the marketplace-health analysis before a commercial decision. | The implied-bump product gap flagged in current-state §3.3. (§7) |
| 19 | **What is actual slot occupancy by tier** in the legacy system? Determines whether full-utilization messaging holds below Platinum. | Savings-messaging features. (§8) |
| 20 | **When is the sector-expansion thesis first tested with real spend**, and on what liquidity threshold is the probe judged? | Phase sequencing; validates ADR-0002's "one table, one column" claim. (§3) |
| 21 | **Do CRM, Intelligence, and Engagements remain three bounded contexts**, or consolidate for comprehension-budget reasons? Needs an ADR before module skeletons exist. | Phase 0 solution structure. (§5) |
