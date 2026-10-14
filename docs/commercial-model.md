# Commercial Model — Findings and Derived Requirements

**Status:** Analysis, v0.1
**Date:** 2026-08-06

**Sources:**

- *Charter School Jobs® — Employer Conversion Strategy* (Google Doc, July 2026) — audited pricing, journey, lifecycle plan. Snapshot: [`sources/2026-10-05-employer-conversion-strategy.pdf`](sources/2026-10-05-employer-conversion-strategy.pdf)
- *K12connect Strategy* handwritten notes (dated pages, latest Jan 5 2026) — product vision. Snapshot: [`sources/2026-10-05-k12connect-strategy-notes.txt`](sources/2026-10-05-k12connect-strategy-notes.txt)
- `K12connect_Domain_List.csv` (105 domains) — revealed product and segment intent. Snapshot: [`sources/2026-10-05-k12connect-domain-list.csv`](sources/2026-10-05-k12connect-domain-list.csv)

This document derives the commercial model from those sources. It exists separately from [`blueprint.md`](blueprint.md) because it is *evidence-derived* rather than *decided* — several items below are inferences that need Kent's confirmation, marked **[CONFIRM]**.

---

## 1. What the sources actually establish

### 1.1 Hard pricing facts (Conversion Strategy §5)

| Tier | Annual price | Postings/yr | Implied /mo | Marketing cost/posting | Fair discount |
|---|---:|---:|---:|---:|---:|
| Bronze | $800 | 60 | 5 | $13.31 | ~~5%~~ **20%** |
| Silver | $1,500 | 180 | 15 | $8.32 | ~~10%~~ **20%** |
| Gold | $2,500 | 600 | 50 | $4.16 | ~~15%~~ **20%** |
| Platinum | $4,000 | 2,400 | 200 | $1.66 | 20% |

> ✅ **Fair discount corrected and resolved 2026-08-06.** The tiered 5/10/15/20% ladder comes from the Conversion Strategy and is **an error in that document**. Both the live site and the internal *Charter School Jobs Price Sheet* show a **flat 20% on every tier**. Commercial consequence stands: the discount gives no reason to upgrade tiers. See [current-state.md §3.2](current-state.md#32-2-is-resolved--the-discount-is-flat-20).

> ✅ **À la carte schedule retrieved 2026-08-06.** 30-day: $100 at qty 1 falling to $50 at qty 25 across nine bands. 60-day: exactly **1.5×** the 30-day price at every band — which makes 60-day credits 25% cheaper *per posting-month* at every band. Measured in posting-months, **break-even against a plan is one continuously-open role**; at two or more concurrent roles the plan wins by 37% and rises to 96% at 200. See [current-state.md §3.3](current-state.md#33-the-plan-vs-à-la-carte-economics--corrected).

Arithmetic check: $800 ÷ 60 = $13.33, $1,500 ÷ 180 = $8.33, $2,500 ÷ 600 = $4.17, $4,000 ÷ 2,400 = $1.67. The published figures are consistent to a cent of rounding. The model holds together.

### 1.2 The single most important structural finding

> ⚠️ **SUPERSEDED 2026-08-06 by [current-state.md §1](current-state.md#1-the-entitlement-model--my-earlier-reading-was-wrong-twice).** This section framed the entitlement as *monthly allowance vs. annual pool*. The live site shows it is **neither** — plans grant **renewable capacity ("slots") with 30-day occupancy**, not a consumable balance. The "rollover" question below is dissolved, not answered. The section is retained because the reasoning it triggered was productive and the seasonality concern survives in modified form. **Do not model against this section.**

The Conversion Strategy quotes the live plans page as selling **"5 job posting credits per Month"**, while the comparison table expresses the same tier as **60 postings/year**.

These are not the same entitlement. A monthly allowance that resets is a fundamentally different object from an annual pool that draws down:

- **Annual pool:** an org can post 60 jobs in March if it wants. Balance is a stock.
- **Monthly allowance:** an org can post 5 jobs in March and 5 in April, and unused March credits vanish. Balance is a flow with a periodic reset.

The lifecycle messaging in the strategy doc — *"you used all 5 slots this month 3 months running — Silver saves you $X"* — only makes sense as a **recurring monthly cap that does not roll over**. That is the reading I've modeled against. **[CONFIRM]**

**Why this matters more than it looks.** Charter hiring is intensely seasonal (the doc puts the peak at Feb–Aug). A monthly non-rolling allowance is structurally hostile to seasonal hiring: a school that posts nothing from September to January and then needs 30 roles in March gets 5. If that's the current behavior, it is very likely a live source of à la carte overage revenue *and* of customer frustration, and the rebuild is the moment to decide deliberately whether to keep it. This is a business decision, not a technical one, but the entitlement model must be able to express whichever answer you choose.

**What survives this correction:** less than the earlier draft claimed. The binding constraint is **concurrency**, and the tier ladder exists precisely to price it — organizations buy the tier covering their peak. The residual is that peak-sizing carries idle off-season capacity, which is normal for reserved-capacity products. See [current-state.md §1](current-state.md) and [§3.3](current-state.md#33-the-plan-vs-à-la-carte-economics--corrected).

**A methodological note worth keeping.** Both this error and the Conversion Strategy's tiered-discount error (see [current-state.md §2](current-state.md)) came from reading **pricing cards** instead of **mechanism copy**. Marketing surfaces describe entitlements in the vocabulary of the buyer, not the system. For the rest of this rebuild, the mechanism is the source of truth and the pricing card is a derived view.

### 1.3 The headline per-posting figures assume 100% utilization

> ❌ **RETRACTED 2026-08-06.** This section is wrong. It measured in *postings per year* when the product is sold in *concurrent capacity*, and it treated full slot utilization as unrealistic. Because CSJ's customers advertise year-round (teacher shortage, high turnover including non-instructional staff), full utilization is the **normal** case. Bronze at full use is $800 ÷ (5 slots × 12 months) = **$13.33 per posting-month** — the published $13.31. The figures are accurate, not aspirational. The "honest numbers will contradict the marketing" warning below is backwards: computed correctly, the lifecycle feature will *confirm* the pricing and show larger savings than the table claims. See [current-state.md §3.3](current-state.md#33-the-plan-vs-à-la-carte-economics--corrected). **Do not model or message against this section.**

Platinum's $1.66 assumes a school posts 200 jobs a month, every month. No charter network does this. The published cost-per-posting numbers are therefore aspirational ceilings, not realized prices.

That's a marketing choice and not mine to make — but it has a direct architectural consequence. The Conversion Strategy proposes building exactly the feature that will expose it:

> *"You spent $X on one-off postings last year — here's what a plan would've saved you"* (Sep–Oct)
> *"you're 3 postings in — a plan would already have paid for itself"* (Mar–May)

These compute against **actual** usage. Built honestly, they will produce numbers that contradict the pricing table. Worth resolving the messaging before building the feature, not after.

### 1.4 Revealed product and segment intent (domain list)

105 owned domains cluster into groups that go well beyond the stated business lines:

| Cluster | Domains | Implication |
|---|---|---|
| **Sector expansion** | `publicschooljobs`, `independentschooljobs`, `magnetschooljobs`, `prepschooljobs`, `prekjobs`, `teacherjobs` | Expansion is by **school sector**, not just geography |
| **Certification** | `certifiedk12` (×4), `certifiedteacherjobs` (×4), `k12certified`, `k12certifiedjobs` | The Jan 2026 CertifiedK12 concept is real and domain-backed |
| **Temp / contract staffing** | `charterschooltemps`, `sub2perm` | **A revenue line absent from the stated business list** |
| **Candidate-side products** | `k12teacherportfolio`, `k12cv`, `mycv` | Free candidate tools — supply-side acquisition and profile data, **not** revenue (see §2.1) |
| **Vendor marketplace** | `charterschoolvendors` | Third-party sellers — matches the "Amazon for 3rd party sellers" note |
| **Media / content** | `charterschoolguide`, `publicschoolguide`, `charterschooldaily`, `charterschoolweekly`, `charterschoolblogs` | Audience products; implies advertising revenue |
| **Recruiting services** | `viralrecruiter`, `viralheadhunter`, `hqteachers` | Existing search/recruiting line |
| **Fairs across sectors** | `k12jobfair(s)`, `publicschoolcareerfairs`, `charterschooljobfair(s)` | Fairs replicate per sector |
| **Platform** | `k12connect` (×8, incl. `.ai`, `.app`, `.io`) | Platform/corporate brand |

**Two things this settles.** First, the expansion axis is **sector** (charter → magnet → pre-K → independent → public) at least as much as geography — same region, different market. Second, **temp/contract staffing** appears nowhere in the stated business description but has dedicated domains; if it's live or planned it is a materially different commercial shape (recurring placement billing, timesheets, markup) and needs to be on the record.

### 1.5 Vision documents — the arc

The handwritten notes trace a clear and useful arc:

- **Expansive** (pages 2–3): "LinkedIn of the education industry" / "Amazon for 3rd party sellers" — profiles, job board + Glassdoor, networking, events, PD, curriculum materials, ATS.
- **Self-correcting** (page 4, 2025-26): *"Need to focus on one clearly defined product that solves a specific problem — 'How do I cost-effectively staff my school with the right-fit, qualified staff?' → Build a hiring machine."*
- **Divergent again** (pages 5–6, Jan 2026): CertifiedK12 as a separate niche business.

Page 4 is the useful one and it already matches the project brief's "Teacher Hiring Machine." I've treated it as the governing statement of focus and everything on pages 2, 3, 5, and 6 as a portfolio of adjacent options — which the architecture should *permit* without *anticipating*.

The concrete test I'd apply to any of them: does it enrich the Organization/Person graph, or does it fork it? Vendor marketplace, PD, and certification all enrich it. Curriculum materials forks it. That's a cheap heuristic for deciding what belongs on this platform.

---

## 2. Corrected revenue shapes

`blueprint.md` §3.4 listed five shapes and assumed throughout that **the employer is the payer**. That assumption is right most of the time and wrong in a way that matters. The corrected list:

| # | Shape | Example | Payer | Structural characteristic |
|---|---|---|---|---|
| 1 | **Periodic allowance subscription** | Annual posting plan | Employer | Annual term, *monthly* grant, drawdown, reset. Not a simple pool. |
| 2 | **Transactional** | Single posting credit | Employer | One-time; durable balance; priced by posting duration (30/60-day) |
| 3 | **Event-scoped** | Career fair table | Employer | Dated event, capacity-limited, price modified by subscription tier |
| 4 | **Campaign** | Employer/candidate marketing | Employer | Time-boxed, deliverables, performance metrics |
| 5 | **Service engagement** | Recruiting, executive search | Employer | Long-running, milestone or placement-based, contingency possible |
| 6 | **Advertising / sponsorship** | Ed schools & PD providers advertising to candidates | **Third party** | Payer is neither employer nor candidate; placement/impression-based |
| 7 | **Metered verification** | "Charge fee to schools to verify" certification status | Employer | Per-query usage billing against a proprietary data asset |
| 8 | **Sponsored candidate services** | Ed school or career services office outsources certification support | **Ed school / institution** | Payer is an institution; the *beneficiary* is a Person |
| 9 | **Contract staffing** *(if real)* | `charterschooltemps`, `sub2perm` | Employer | Recurring placement billing, markup, conversion fees. **[CONFIRM]** |

> **Corrected 2026-08-06.** An earlier draft listed a *candidate-paid service* shape. That was an inference from the `k12cv` / `mycv` / `k12teacherportfolio` domains and the CertifiedK12 notes, and the sources do not support it — *"maybe we could build this to attract candidates"* is acquisition language, and the only revenue mechanisms named in those notes are advertising and school-paid verification. **CSJ does not charge candidates.** See §2.1.

### 2.1 Constraint: candidates never pay

This is a standing business commitment, not an observation about the current catalog. It is recorded here because it is load-bearing:

- The candidate side is **acquisition and data**, never revenue. Its value is realized indirectly — employers pay for access to the audience, advertisers pay for attention within it.
- Candidate-facing products (CV builder, portfolio, certification pathway guidance) are therefore **free tools whose return is supply-side liquidity and profile data**, and should be evaluated on candidate acquisition and profile completeness, not on revenue.
- Where a candidate-benefiting *service* is monetized, the payer is an institution — a school verifying credentials, or an ed school outsourcing certification support (shape 8). The candidate benefits; someone else pays.

It's worth writing down *why*, because a future reader looking at a CV product will otherwise be tempted to put a price on it: charging job seekers in a mission-driven K–12 market is reputationally corrosive and would compromise the candidate-supply advantage the whole platform depends on. If this ever changes it should change deliberately, via an ADR.

### The payer correction — restated on stronger grounds

The candidate case is withdrawn, but **Order still needs a Payer Party distinct from a Beneficiary Party**, and the argument is better without it — because the split already exists in today's core business.

**The current-business case (strongest):** Platinum grants 2,400 postings a year. No single school posts at that volume; that tier only makes sense for a **network or CMO buying on behalf of its member schools**. So the question "who paid?" and "whose postings are these?" already have different answers, today, with no reference to candidates or new product lines. The blueprint's assumed `Order → Organization` cannot express it.

> **Corrected 2026-08-06.** An earlier version of this paragraph added that Payer/Beneficiary is what answers the Conversion Strategy KPI *"average individual-credit spend per school,"* on the grounds that *per school* means the beneficiary. **That reasoning is wrong under pooling** (see below). For a pooled network purchase there is no single member school to record as beneficiary — the beneficiary *is* the network. Per-school attribution comes from consumption (`Publication → JobOpening → Organization`), not from the Order. The Payer/Beneficiary split is still required; this particular argument for it is not.

**The strongest case is instead the variability itself.** For some clients the network is the customer of record; for others each school is; and for at least one, it varies *from purchase to purchase* because the client's own management is unsettled. That is per-transaction variance, which only per-Order fields can represent. A single `Order → Organization`, or an Organization-level "customer of record" flag, forces one answer per client and is silently wrong the rest of the time.

**Hierarchy scoping — resolved 2026-08-06.** An entitlement purchased by a parent Organization and consumed by children is a hierarchy-scoped entitlement. Networks **do** buy centrally, and the allowance is **pooled across the network**, not allocated per school. Any member school draws from the shared pool. Modelled as `Entitlement.HolderOrganizationId` plus a `ConsumptionScope` of `Self | SelfAndDescendants | ExplicitList`, evaluated against the effective-dated hierarchy at the moment of consumption. See **[ADR-0008](adr/0008-products-plans-and-entitlement.md) §4**.

**The supporting case:** shape 8 (institution pays, Person benefits) and shape 7 (school pays, the subject of the transaction is a Person).

**Resolution, unchanged and still cheap:** `Order.PayerPartyId` → `Party`, `Order.BeneficiaryPartyId` → `Party`, **both non-null, Beneficiary defaulting to Payer at insert**. In the overwhelmingly common case both point at the same Organization. Non-null avoids a `COALESCE` on every consuming query, which is where this design would otherwise leak silent bugs into revenue reporting. The Party abstraction in blueprint §4.2 absorbs it at no cost, including shape 8 where the beneficiary is a Person.

**Which one to use, mechanically:** money questions use **Payer** (revenue, invoicing, AR, collections, tax); everything else uses **Beneficiary** (entitlement, utilization, account management, CRM, lifecycle KPIs). See [ADR-0008 §1](adr/0008-products-plans-and-entitlement.md).

### The advertiser correction

Ed schools and PD providers buying advertising are **Organizations with role = Advertiser** — a role missing from the blueprint glossary. It should be added alongside Employer, Client, Prospect, Partner, Sponsor. No structural change needed; the effective-dated Organization Role design already handles it. Worth noting as evidence the role model is right.

---

## 3. Entitlement — corrected model

Blueprint §3.4 defined Entitlement as "quantity + validity window." There are **four structurally distinct kinds**, and collapsing them is the mistake to avoid.

> **Revised 2026-08-06.** An earlier draft listed three kinds and modelled plans as a consumable monthly allowance. The live site shows plans grant **renewable capacity**, which is a fourth and different kind. See [current-state.md §1](current-state.md#1-the-entitlement-model--my-earlier-reading-was-wrong-twice).

### 3.1 Renewable capacity — "slots" *(the annual plans)*

N concurrent slots. A posting occupies a slot for 30 days, then the slot is automatically released and reusable. **Not a balance that depletes** — a capacity that is temporarily occupied.

```
Subscription (Bronze, annual term 2026-09-01 → 2027-08-31)
  └── SlotEntitlement (capacity = 5)
        ├── SlotOccupancy (posting #1041, 2026-09-03 → 2026-10-03)   ← releases automatically
        ├── SlotOccupancy (posting #1042, 2026-09-07 → 2026-10-07)
        └── ...                                        3 of 5 slots free on 2026-09-08
```

Occupancies are **materialized records, not computed** — for the same reasons the earlier draft gave, which survive the correction: utilization is auditable, mid-term tier changes are expressible, the Conversion Strategy's "what did you actually use" analytics become a simple query, and a policy change applies forward without rewriting history.

The question this model must answer constantly is *"can this Organization publish right now?"* — which is a **concurrency check against current occupancy**, not a balance lookup. Very different query, very different index.

### 3.2 Consumable credit *(à la carte)*

A durable balance, drawn down permanently, SKU'd by duration (30-day / 60-day) and bought in quantity bands with volume pricing.

This is the model the earlier draft mistakenly applied to plans. It's correct here — and the two coexisting is the substantive finding: **an Organization may hold both capacity and credits simultaneously**, and the platform must resolve both when answering "can this org post?"

**Consumption order — decided 2026-08-06.** *Perishability first, then proximity*: slot capacity before credits, and within each, the publishing Organization's own before an in-scope ancestor's. Slot capacity is use-it-or-lose-it within the subscription term; a credit is durable and never expires, so spending the perishable resource first is strictly better for the customer. This is CSJ policy rather than an external fact, so it was decided rather than researched — see [ADR-0008 §5](adr/0008-products-plans-and-entitlement.md). Reversing it is a new ADR, not a code edit.

### 3.3 Non-consumable benefit

The career fair discount is **not a quantity**. It is a pricing modifier attached to the subscription tier, applying to a *different product line*, valid while the subscription is active.

This is the one the blueprint's definition couldn't express at all — and per the Conversion Strategy, it's also the most commercially important underused lever in the business ("the connective tissue"). It needs to be a first-class entitlement kind, because §4 of that document proposes surfacing it at six different points in the funnel, each of which needs to ask the same question: *what discount does this org get on this fair table, right now?*

Note that the live rate is a **flat 20% across all tiers**, not the tiered ladder the Conversion Strategy assumes ([current-state.md §2](current-state.md#2-contradiction-the-career-fair-discount-is-not-tiered)). The entitlement kind expresses either equally well; the business question is unresolved.

### 3.4 Capability flag

From Strategy page 1 — ATS-like functionality (contact, scheduling, candidate management, AI JD copilot, credential verification, HRMS export) delivered *inside* the posting experience. Whether a given posting has these depends on what was paid.

### 3.5 The unifying rule

> **Entitlement resolution is one service. Nothing else in the platform asks "does this org have a plan?"**

Every access check, price calculation, and feature gate calls it. This is the single most valuable constraint in the commercial model, because entitlement logic leaking into features is the standard way platforms like this become unmaintainable.

---

## 4. Posting is not a fungible unit

The blueprint implicitly treated a Posting as one unit consumed from a pool. The sources contradict this twice:

1. **Duration is priced.** The Conversion Strategy notes the à la carte page has "30-day/60-day" rates. Does a 60-day posting consume two plan credits, or one? **[CONFIRM]** — this is a real business rule with no obvious default.
2. **Capability varies by what was paid.** Strategy page 1 describes a "full-price posting" carrying ATS functionality. A posting effectively costing $1.66 under Platinum cannot reasonably include the same depth as a premium à la carte posting.

**Therefore:** Posting references a **PostingProduct** (SKU) that determines duration, price, and included capabilities. Postings are not interchangeable, and the entitlement consumed depends on the SKU.

This also resolves the ATS tension productively — see §6.

---

## 5. Requirements the lifecycle plan imposes on the data model

Conversion Strategy §6 (marketing calendar) and §7 (KPIs) are not just marketing; they are a specification for what the commercial data must be able to answer.

| Source requirement | Data model consequence |
|---|---|
| *"You spent $X on one-off postings last year — here's what a plan would've saved you"* | **Counterfactual pricing.** Requires immutable price-and-terms snapshots on every order line. Never resolve historical price by lookup against the current catalog. |
| *"% of fair purchases made by existing plan holders"* | **Point-in-time subscription state.** Requires effective-dated subscription history, not a current-tier column. |
| *"Plan tier upgrade rate at renewal"* | Subscription **term and transition history** as first-class records — renewals, upgrades, downgrades, lapses are events, not field updates. |
| *"Average individual-credit spend per school before plan conversion"* | Organization-level **lifetime commercial history**, spanning products, with conversion events identifiable. *Per school* resolves through **consumption** (`Publication → JobOpening → Organization`), not through `Order.BeneficiaryPartyId` — under pooling the beneficiary is the network. |
| *"You used all 5 slots this month 3 months running"* | **Per-period utilization** is queryable — which §3.1's materialized grants give for free. |
| *"Growth Bundle" (plan + prepaid fair table as one SKU)* | **Composite products**: one order line grants entitlements across product lines. |
| Fair table price varies by buyer's tier | **Contextual pricing**: price of product B depends on the buyer's active entitlements at time of quote. |

Every one of these is satisfied by the same underlying discipline — *preserve what was true at the time, don't overwrite it* — which is already platform principle #4. The commercial domain turns out to be the strongest justification for it.

---

## 6. Conflicts with the current blueprint

Per the documentation rules, conflicts are named before they're resolved.

### Conflict 1 — the ATS non-goal *(material)*

**Blueprint §1.3 says:** "Not a full ATS. CSJ is not competing with Workday or Greenhouse on applicant workflow depth."

**Strategy page 1 says:** *"Imagine a product where a full-price job posting would have all the functioning of a sophisticated (but simple to use) applicant tracking system built into the posting experience"* — contact system (email, text, voice), scheduling, candidate management, AI JD copilot, AI candidate search, state certification database integration, HRMS export, background check integration.

**Assessment:** I wrote the non-goal without this source and it is too broad. But the vision note is also subtler than "build an ATS," and the distinction is worth preserving. The idea is *the posting is the ATS* — ATS capability is the value delivered inside a product you already sell, not a separately licensed system. That is a packaging insight, and a genuinely good one: it sidesteps competing with Greenhouse on features while raising the value of the core SKU.

**Proposed revision:**

> **Not a standalone ATS product.** We do not sell seat-licensed applicant tracking, and we do not build offer management, onboarding, or HRIS. We *do* build applicant workflow depth into the posting experience itself, gated by posting SKU — because the posting is the product.

This keeps the useful constraint (no HRIS, no offer management, no onboarding) and removes the false one.

### Conflict 2 — revenue shapes and payer *(material)*

Blueprint §3.4 lists five shapes and assumes the employer pays. Corrected to nine shapes with an explicit Payer/Beneficiary split (§2 above).

### Conflict 3 — entitlement definition too narrow *(material)*

Blueprint defines Entitlement as quantity + validity window. Corrected to **four** kinds (§3 above), each with a holder and a consumption scope. Settled in [ADR-0008](adr/0008-products-plans-and-entitlement.md).

### Conflict 4 — Posting treated as fungible *(material)*

Corrected in §4 above; Posting requires a SKU.

### Conflict 5 — "Not a real-time system" *(minor)*

Strategy page 1 contemplates text and voice contact plus scheduling. Neither strictly requires real-time infrastructure — both are fine as queued/webhook integrations with third-party providers. The non-goal stands, but it should say "no real-time infrastructure of our own; real-time-feeling features are delegated to providers."

### Conflict 6 — ADR-0002 framing *(material)*

See §7.

---

## 7. Answering blueprint open question #1 — what is K12connect?

**Neither a market nor a tenant. K12connect is the corporate/platform brand.**

Evidence:

- Kent's own email is `@k12connect.com`; the domain family includes `.ai`, `.app`, `.io`, `.com`, `.net`, `.org`, `.co`, `.info` — a defensive corporate spread, not a product launch.
- Every vision note describes K12connect as *the platform* ("The K12connect system", "K12connect as the LinkedIn of the education industry"), with Charter School Jobs® as a brand operating on it.
- The expansion described in the notes is **product-line** expansion (certification, PD, events, vendors, profiles), not geographic expansion.

**Consequent three-layer structure:**

```
K12connect                     — platform / corporate brand
  ├── Charter School Jobs®     — market-facing brand, charter sector
  ├── [public / magnet / independent / pre-K]  — same product, different sector  (domains owned)
  └── CertifiedK12             — distinct product, shares the Organization + Person graph
```

**Impact on ADR-0002:** the recommendation (single instance, `Market` scope, `Brand` as presentation) survives intact and is *strengthened* — CertifiedK12's value proposition ("proprietary database... we guarantee accuracy") depends entirely on a single unforked Person graph. Any tenancy option would destroy it.

But two refinements are needed:

1. **Market is a *sector* dimension at least as much as a geographic one.** `charterschooljobs` / `publicschooljobs` / `magnetschooljobs` / `prepschooljobs` / `prekjobs` are the same region and the same product, sold to different school sectors. Since an Organization's sector is an attribute of the Organization, a Market may be *definable* as a query over Organizations rather than a hard scope on them — which is cheaper and more flexible than I assumed. Worth exploring before finalizing.
2. **Product line is a real dimension and it isn't tenancy.** CertifiedK12 shares Organizations and People but has its own catalog, its own payer types (candidates, advertisers), and its own delivery model. That's handled by the Commerce context, not by a scoping mechanism — which is a genuinely good outcome, because it means product expansion costs nothing architecturally.

### The one thing that would change this answer

If CertifiedK12's "credit-reporting-agency" positioning ever means selling verification data *about* candidates to parties with no relationship to them, that raises consent, privacy, and possibly FCRA-adjacent regulatory questions that are a different class of problem from anything else here. Worth a deliberate look before it's built — I'd want it scoped as its own decision rather than absorbed into the platform by default.

---

## 8. Open questions for Kent

Ordered by how much they block Phase 1.

1. **Do plan credits reset monthly, or pool annually?** Do unused credits roll over? Blocks the entitlement model entirely. (§1.2)
2. **Do networks/CMOs buy plans centrally for member schools?** If so, is the allowance pooled across the network or allocated per school, and who controls the split? Platinum's 2,400 postings/year strongly implies central purchasing. Blocks both the entitlement model and organization hierarchy. (§2)
3. **Does a 60-day posting consume one credit or two?** Blocks Posting/SKU modeling. (§4)
4. **What is the subscription term anchored to** — calendar year, school year (Jul 1), or purchase anniversary? Determines renewal clustering, proration, and how the Nov–Dec budget-season campaign actually works.
5. **Is temp/contract staffing (`charterschooltemps`, `sub2perm`) live, planned, or abandoned?** It's the one revenue shape with domains but no mention in the business description, and it's structurally unlike the others. (§1.4)
6. **What is the current à la carte credit price?** The Conversion Strategy flags it as unretrieved (rendered via JS). Needed to complete the pricing model — and it's the anchor for the whole cost-comparison argument.
7. **Are career fair discounts applied at purchase, or invoiced as a rebate?** Determines whether this is a pricing-engine concern or a billing concern.
8. **Is the "Growth Bundle" a real near-term SKU?** If yes, composite products enter Phase 2 rather than later.
9. **What happens on overage today** — blocked, or auto-charged at à la carte rates? This is likely a meaningful revenue line and it's entirely undocumented.
