# ADR-0002: Tenancy and market scope

- **Status:** Proposed — awaiting decision
- **Date:** 2026-08-06
- **Updated:** 2026-08-06 — revised after analysis of the Strategy documents
- **Deciders:** Kent Strong
- **Related:** ADR-0001, [commercial-model.md](../commercial-model.md)

## Update — evidence from the Strategy documents

Analysis of the K12connect strategy notes and the 105-domain portfolio (see [commercial-model.md §7](../commercial-model.md#7-answering-blueprint-open-question-1--what-is-k12connect)) resolves the central open question and changes the framing in two ways. The **recommendation below is unchanged and strengthened.**

**1. K12connect is the platform/corporate brand — not a market, not a tenant.** The structure is three layers:

```
K12connect                     — platform / corporate brand
  ├── Charter School Jobs®     — market-facing brand, charter sector
  ├── [public / magnet / independent / pre-K]  — same product, other sectors
  └── CertifiedK12             — distinct product line, shares Org + Person graph
```

**2. Market is a *school-sector* dimension at least as much as a geographic one.** The owned domains include `publicschooljobs`, `magnetschooljobs`, `independentschooljobs`, `prepschooljobs`, and `prekjobs` — same region, same product, different school sector. This is a materially different (and cheaper) shape than geographic expansion.

**3. Product line is a real dimension, and it is not tenancy.** CertifiedK12 shares Organizations and People but has its own catalog, its own payer types (candidates, advertisers), and its own delivery model. That is entirely a Commerce-context concern. Product expansion therefore costs *nothing* architecturally — which is the strongest possible argument against building a scoping mechanism for it.

**Why this strengthens Option B:** CertifiedK12's stated value proposition — *"we keep a proprietary database sorta like credit reporting agencies where we guarantee accuracy of candidate's certification status"* — depends entirely on a single, unforked Person graph with reliable provenance. Every isolation option destroys it. The commercial vision and the data architecture point the same direction.

**New consideration raised (see §Open questions):** if Market is a school-sector attribute of an Organization, a Market may be *definable as a query over Organizations* rather than a hard `MarketId` scope on them. That would be cheaper and more flexible than the recommendation below assumes, and should be settled before implementation.

## Context

CSJ serves charter schools primarily in Metro New York. The stated long-term direction is a comprehensive K–12 hiring platform, and K12connect exists as a separate brand with its own owned domains (`K12connect_Owned_Domain_List.csv` in the repository root).

"Tenancy" is being used loosely, and three genuinely different things are hiding under the word. They must be separated before deciding anything:

1. **Market** — a geographic or sector-scoped job marketplace (Metro NY charter schools; later, e.g., Texas charters, or K–12 broadly). Markets share a candidate pool conceptually but present different sites.
2. **Brand** — the public identity a market is presented under (Charter School Jobs®, K12connect). A brand may span markets; a market may be presented under more than one brand.
3. **Tenant (isolation)** — a hard data boundary, typically because a customer demands their data be separate. Relevant only if CSJ ever sells a white-labeled private job board to a large CMO or authorizer.

These are independent. Conflating them is the most common way this decision goes wrong.

## Decision drivers

- Reversibility. Retrofitting isolation into a shared model is expensive; removing unnecessary isolation later is merely tedious. Asymmetry favors caution, but not paranoia.
- Solo development with AI-assisted implementation. Every axis of variability multiplies the cost of every feature. A tenancy dimension threaded through all queries is a permanent tax on velocity.
- Actual scale. Metro NY charter sector is small — plausibly hundreds of organizations, low thousands of postings per year, tens of thousands of candidates. No scale pressure justifies isolation.
- Data value. The commercial asset CSJ is building is a *connected* dataset of organizations, contacts, and hiring signals. Hard tenant isolation fragments exactly the asset that makes the platform valuable.

## Options

### A. Single instance, no tenancy concept

One deployment, one database, no scoping dimension. Markets and brands, if they arrive, are handled by adding site-level content and filters at that time.

- **For:** Simplest possible model. Zero ongoing tax.
- **Against:** If a second market arrives, retrofitting scope into every query and every uniqueness constraint is a large, error-prone migration.

### B. Single instance, explicit Market scope on relevant entities (recommended)

One deployment, one database, one Orchard Core tenant. A `Market` entity exists from day one. Entities that are inherently market-scoped (job postings, career fairs, subscriptions, site content) carry a `MarketId`. Entities that are inherently *global* (Organization, Person, external identifiers, enrichment data) deliberately do **not**.

Brand is a presentation concern — a `Brand` record with domain bindings, resolved by hostname, mapping to one or more markets. K12connect domains bind to a brand, not to a separate deployment.

- **For:** The expansion path is open without paying isolation costs. The connected data asset stays connected — one Organization record, however many markets it recruits in. Cheap to build now (essentially one table, one column on a handful of entities). Brand/domain routing is a solved problem in Orchard Core.
- **Against:** Requires discipline about which entities are scoped and which are global — get this wrong and you either fragment the dataset or leak data across markets. No hard isolation guarantee; a query bug can cross markets.
- **Note:** Deciding *now* which entities are global is the entire value of this option. That analysis is cheap today and expensive in two years.

### C. Orchard Core multi-tenancy (tenant per market/brand)

Each market runs as an Orchard Core tenant, with its own content and — depending on configuration — its own database or schema.

- **For:** Strong isolation. Independent content, theming, and configuration per market, natively supported.
- **Against:** Fights ADR-0001 directly. Orchard tenancy scopes the *Orchard* side cleanly — the documentation describes "database, content, theme and user isolation" per tenant — but that isolation covers Orchard's own stores, **not an EF Core `DbContext` we bring ourselves**. Making a custom `DbContext` tenant-aware has been raised as a feature request against the project ([OrchardCMS/OrchardCore#1343](https://github.com/OrchardCMS/OrchardCore/issues/1343)), which is itself evidence it isn't handed to you. We would own per-tenant connection resolution, and cross-tenant domain queries (which marketing intelligence absolutely needs) become genuinely hard. *Verify the current state of that issue before relying on this point.* Shared Organization/Person data across tenants requires a separate shared store, which is the complexity of option B *plus* the complexity of tenancy. Operationally heavier: migrations per tenant, backups per tenant.
- **Verdict:** Wrong for markets. Potentially right for a future white-label product, which is a different problem.

### D. Separate deployments per brand

CSJ and K12connect as independent applications sharing code.

- **For:** Total isolation, independent release cadence.
- **Against:** Duplicated operations, duplicated data, and the organization dataset — the crown jewel — gets forked. For a solo developer this is disqualifying.

## Recommendation

**Option B.** Single instance, single Orchard tenant, explicit `Market` scope on market-bound entities, `Brand` as a hostname-resolved presentation layer, and a deliberate, documented list of globally-shared entities.

Rationale: the asset CSJ is building is the connected organization/contact/signal graph. Every isolation option fragments it. Option B keeps the graph whole while making geographic expansion a configuration change rather than a rewrite, and it costs perhaps a day of extra modeling now.

If a white-label private job board is ever sold, treat that as a genuinely separate product decision (a new ADR), not as a reason to pre-build tenancy today.

### Proposed classification

**Global (never market-scoped)**

- Organization, organization hierarchy, external identifiers, enrichment/provenance records
- Person/Party and identity resolution
- Product catalog definitions

**Market-scoped**

- Job postings, applications
- Career fairs and registrations
- Subscriptions, orders, and entitlements
- Site content, navigation, landing pages

**Brand-level (presentation only)**

- Domains, theming, email templates, logos, legal footer

### If accepted, the implementation implications are

- `Market` and `Brand` tables in a shared schema, seeded with a single market ("Metro NY Charter") from day one so the code path is exercised immediately.
- `MarketId` is non-nullable on scoped entities. No nullable "means all markets" escape hatch — that ambiguity metastasizes.
- A single, centrally enforced market filter (EF Core global query filter) rather than per-query `WHERE` clauses.
- Uniqueness constraints on scoped entities include `MarketId`.

## Open questions for Kent

1. ~~Is K12connect a brand, a market, or a distinct product?~~ **RESOLVED** — platform/corporate brand. See the Update section above.
2. **Should Market be a hard scope (`MarketId` column) or a derived query over Organization sector?** If sectors are mutually exclusive and stable per Organization, the derived approach is cheaper and avoids the classification discipline Option B otherwise requires. If an Organization can participate in two markets — plausible for a CMO operating both charter and public schools — the hard scope wins. **This is now the deciding question for this ADR.**
3. Is there any realistic near-term prospect of selling a private/white-labeled job board to a large CMO or authorizer? Still the only scenario that would push toward Option C.
4. Are the sector domains (`publicschooljobs`, `magnetschooljobs`, etc.) planned properties or defensive registrations? Determines whether sector expansion is a Phase 5 concern or nearer.
