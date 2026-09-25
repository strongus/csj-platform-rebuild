# ADR-0008: Products, plans, and entitlement

- **Status:** Proposed
- **Date:** 2026-08-06
- **Deciders:** Kent Strong
- **Related:** ADR-0004 (organization definition and hierarchy — **upstream, blocking**), ADR-0012 (price schedules and contextual pricing), ADR-0009 (authorization), [commercial-model.md §2–§4](../commercial-model.md), [current-state.md §1](../current-state.md), blueprint §4.1–4.2, §5

## Context

Revenue arrives in nine structurally different shapes ([commercial-model.md §2](../commercial-model.md)) and entitlement comes in four mechanically distinct kinds (§3). Two questions have been carried as open items and both are now answerable:

**1. Who is the customer?** Platinum grants 2,400 postings a year. No single school posts at that volume — that tier only makes sense for a network or CMO buying on behalf of its member schools. "Who paid?" and "whose postings are these?" therefore already have different answers in today's business.

The sharpest case is a client whose management is genuinely ambiguous: for some purchases the network is the customer of record, for others an individual school is. This is not a data-quality problem to be cleaned up. **It varies per transaction, and the model must be able to say so.**

**2. How do network-bought postings get used?** Confirmed 2026-08-06: **pooled across the network.** Any member school draws from a shared pool. Networks do not pre-allocate slots per school. This resolves the `[CONFIRM]` at [commercial-model.md §2](../commercial-model.md).

The pooling answer changes an assumption in the existing text. If the pool is shared, then a network's purchase has no single school beneficiary, and per-school utilization cannot come from the Order at all — it comes from which school's Opening consumed the slot.

Finally, invariant §3 in `CLAUDE.md` requires that entitlement resolution live in exactly one service. That constraint is only enforceable if there is a single, explicit definition of what an entitlement is held by and who may consume it. This ADR supplies it.

## Options

### A. Order references a single Organization

**For.** One FK. No "which one do I join on?" decision on any query. Matches the 95% case where payer and beneficiary coincide. Lowest cognitive load, which is the binding constraint under the right-sizing rule.

**Against.** Cannot represent a network buying for members, which exists today. Cannot represent shape 8 (institution pays, Person benefits). The ambiguous-management client forces a single wrong answer per organization, so the real relationship survives only in a note field or in the operator's head. Breaks the Conversion Strategy KPI *"average individual-credit spend per school."*

**Verdict.** Rejected. It fails against current business, not hypothetical business.

### B. Organization-level "customer of record" flag

**For.** Keeps one FK on Order and records the network-vs-school relationship once.

**Against.** Wrong cardinality. The relationship varies **per transaction**, not per organization. Any flag makes the ambiguous-management client wrong roughly half the time, and wrong silently.

**Verdict.** Rejected.

### C. Payer and Beneficiary Party references on Order, Beneficiary nullable

**For.** Expresses everything needed. Null signals "same as payer," so the common case is visibly unremarkable.

**Against.** Every consuming query needs `COALESCE(BeneficiaryPartyId, PayerPartyId)`. One developer will eventually forget it, and the resulting bug is silent and in a revenue report.

**Verdict.** Rejected on the null-handling tax alone.

### D. Payer and Beneficiary Party references, both non-null, Beneficiary defaults to Payer

**For.** All of C's expressiveness. No null handling anywhere. Reads identically to option A in the common case. Costs one FK column and one default at insert.

**Against.** Still imposes a per-query decision about which reference to use. Mitigated by a mechanical rule (see Decision §1).

**Verdict.** **Accepted.**

### E. Entitlement scope — pool only

**For.** Matches confirmed current behaviour exactly. Simplest possible implementation.

**Against.** Cannot express a school that buys independently while also belonging to a pooling network — which is precisely the ambiguous-management case. Adding it later means migrating live entitlements.

**Verdict.** Rejected as the *only* mechanism, but accepted as the default.

### F. Entitlement scope — holder plus an explicit scope rule

**For.** Pooling is `SelfAndDescendants`, an independent school purchase is `Self`, and the ambiguous case is `ExplicitList`. All three become **data**, not branching code. One resolution path serves all of them, which is what invariant §3 requires.

**Against.** Three scope kinds where the business currently uses one. `ExplicitList` needs a membership table that may go unused.

**Verdict.** **Accepted.** The added surface is one enum column and one small table, and it converts the messiest client relationship from an exception into a row.

## Decision

### 1. Order carries Payer and Beneficiary

```
Order
  PayerPartyId        → Party   (NOT NULL)  — who is invoiced
  BeneficiaryPartyId  → Party   (NOT NULL)  — who the purchase is for; defaults to Payer
```

Both resolve to `Party`, so shape 8 (institution pays, Person benefits) needs no special case. In the overwhelmingly common case they are the same Organization.

**Disambiguation rule — mechanical, no judgement required:**

> **Money questions use Payer. Everything else uses Beneficiary.**
>
> Revenue, invoicing, AR, collections, tax, dunning → **Payer**.
> Entitlement, utilization, account management, CRM, lifecycle KPIs → **Beneficiary**.

This rule exists to remove a recurring per-query decision. It belongs in `CLAUDE.md` §4, not only here.

**UI:** present one field ("Billed to"). Reveal the second only when it differs from the first.

### 2. Per-school attribution does not come from Order

For a pooled network purchase the beneficiary **is the network**. There is no member school to name at purchase time, because allocation does not happen at purchase time.

Per-school utilization therefore derives from consumption:

```
Publication → JobOpening → Organization
```

This is a better source than the Order regardless of pooling — it reflects what was actually used rather than what was intended, and it survives a school joining or leaving the network mid-term.

### 3. Four entitlement kinds — never collapsed

Per [commercial-model.md §3](../commercial-model.md), restated here only as the field this ADR scopes:

| Kind | Mechanic | Core question |
|---|---|---|
| **Renewable capacity (slot)** | N concurrent occupancies, auto-released after the publication window | Concurrency check |
| **Consumable credit** | Durable balance, depletes permanently, SKU'd by duration | Balance lookup |
| **Non-consumable benefit** | Pricing modifier on another product line | Modifier lookup at quote time |
| **Capability flag** | Feature gate tied to a purchased unit | Boolean at render time |

They share a holder and a scope (§4). They share nothing else. Do not build a common `Quantity` column across them.

### 4. Every entitlement has a holder and a consumption scope

```
Entitlement
  HolderOrganizationId  → Organization   (NOT NULL)
  ConsumptionScope      → enum           (NOT NULL)
                          Self | SelfAndDescendants | ExplicitList
```

| Scope | Who may consume | Use |
|---|---|---|
| `Self` | The holder only | A school buying for itself |
| `SelfAndDescendants` | Holder plus current hierarchy descendants | **Network pool — the confirmed default for network purchases** |
| `ExplicitList` | Holder plus a named set of Organizations | Ambiguous management; a network covering only some members |

**Scope is evaluated against the effective-dated hierarchy as of the moment of consumption**, per blueprint §4.1(b). A school that leaves a network stops drawing on its pool from the departure date forward, without any retroactive effect.

**The resolved holder and scope are written onto the consumption record** (`SlotOccupancy` / credit draw), not re-derived later. Hierarchy changes must not silently rewrite the history of who paid for what — platform principle #4.

### 5. Consumption resolution order

When an Organization publishes and more than one entitlement could serve it, resolution is **deterministic and stated once**, in the entitlement service:

1. Slot capacity held by the publishing Organization
2. Slot capacity held by the nearest ancestor in scope, then successively more distant ancestors
3. Credits held by the publishing Organization
4. Credits held by the nearest ancestor in scope, then outward

**Perishability first, then proximity.** Slot capacity is use-it-or-lose-it within the subscription term; a credit is durable and does not expire. Spending the perishable resource first is strictly better for the customer and removes a recurring support conversation. Proximity breaks remaining ties because the entity closest to the vacancy is the most plausible owner of the budget.

This resolves the consumption-order `[CONFIRM]` at [commercial-model.md §3.2](../commercial-model.md). It is **CSJ policy rather than an external fact**, so it is decided here rather than researched — but it is reversible, and reversing it is a new ADR, not a code edit.

### 6. Plan tiers are data rows

A tier is a row in a plan catalog with effective-dated pricing and grant terms. A negotiated Custom plan is a new row. **Never an enum, never a `switch`.** This is already an invariant in `CLAUDE.md` §3; it is repeated here because the gap above Platinum (blueprint open question #5) will be the first thing that tests it.

### 7. Enforcement

Invariant §3 — "entitlement resolution is one service" — is invisible in code and violating it compiles. It gets the same mechanical treatment as the Orchard boundary:

- Entitlement entity types are `internal` to `CSJ.Commerce`. Outside callers receive resolution **results**, never entitlement rows.
- An architecture test asserts that no assembly outside `CSJ.Commerce` references an entitlement type, and that no assembly outside the entitlement service performs a concurrency count over `SlotOccupancy`.
- Added to `CSJ.ArchitectureTests`, run under `dotnet test --filter Category=Architecture`.

Without this, `if (org.HasPlan)` reappears in the Hiring and Events modules within a release or two.

## Consequences

**Positive**

- The ambiguous-management client needs no special-casing: a network pool and a school's own purchase are simply two entitlements with different holders, resolved in a stated order.
- Pooling, independent purchase, and partial coverage are all the same code path with different data.
- Per-school utilization becomes measurable for the first time, from consumption rather than intent.
- Payer/Beneficiary satisfies shape 6 (third-party advertiser) and shape 8 (sponsored candidate services) with no further modelling.
- Hierarchy changes cannot corrupt commercial history, because consumption records are self-contained.

**Negative / accepted**

- Two Party references on Order impose a per-query decision. Accepted, mitigated by the §1 rule.
- `ExplicitList` may have no member rows at launch. Accepted — the column costs nothing and retrofitting scope onto live entitlements would not be cheap.
- Resolution order is a business policy encoded in a service. It must be documented in customer-facing terms, or support will be asked why a credit was spent.
- The entitlement service becomes a hot path on publish. At CSJ's volumes this is immaterial, and it must not motivate a cache.

**Blocking dependency**

ADR-0004 (what an Organization *is* — legal LEA, operating entity, or campus) is **upstream of this ADR**. Scope evaluation walks the hierarchy, so if the hierarchy's nodes are ambiguous, so is every entitlement check. Blueprint open question #7 notes Great Oaks, Lighthouse Academies, and Citizens of the World each already appear as both network and campus. **ADR-0008 should not be implemented before ADR-0004 is accepted.**

## Open questions

1. Does a 60-day publication occupy one slot for 60 days, or two slots? Existing `[CONFIRM]` at [commercial-model.md §4](../commercial-model.md); a real business rule with no obvious default.
2. What happens to an in-flight `SlotOccupancy` when the consuming school leaves the network mid-publication? Proposed: it runs to expiry and is not renewable from that pool. Needs confirmation.
3. Can a network see, and cap, per-school draw against a shared pool? Pooling without a cap means one school can exhaust the network's capacity. Likely a real customer request; does not change the model, only the reporting and an optional per-school limit.
4. Is the career fair discount flat 20% or tiered? ([current-state.md §2](../current-state.md)) — affects the non-consumable benefit's shape, not its existence.
5. Is there a Custom tier above Platinum? Blueprint open question #5. Decided as a data row either way; the commercial question is separate.
6. Does the ambiguous-management client require `ExplicitList` today, or is `Self` plus `SelfAndDescendants` sufficient with two separate entitlements?
