# CSJ Platform Blueprint

**Status:** Living document — v0.1
**Last updated:** 2026-08-06

This document holds what should not change often: what we are building, the language we use to describe it, the capabilities it must support, and the boundaries between its parts. Volatile decisions — technology choices, security mechanisms, integration specifics, AI approaches — live in [`adr/`](adr/README.md) and are referenced, not restated, here.

---

## 1. Vision, principles, and non-goals

### 1.0 Mission (K12connect, Inc., 2022)

> *We believe educating children is essential to the progression of humanity. This critically important work should be done by the most capable and academically accomplished members of our society. **Our mission is to help K-12 educational organizations attract and hire these talented individuals.***

Three things this settles, and they are load-bearing for the architecture:

1. **The customer is the organization.** "Help K-12 educational **organizations** attract and hire." This is the mission-level basis for the standing constraint that **candidates never pay** (§3.4) — candidates are the talent being attracted, not the customer being served.
2. **The scope is K–12, not charter.** Charter is the beachhead. The mission was already written at K–12 scope in 2022, which is consistent with the sector-expansion domains and with national customers already on the books (§6.1).
3. **The product is match quality, not listing volume.** "The most capable and academically accomplished" is a selectivity claim. A platform that competes on quality of match rather than quantity of postings needs rich Person data, credential verification, and provenance — which is why the Intelligence context and the §4.4 provenance chain are foundational rather than decorative.

The selectivity mandate also carries an obligation. Operationalizing "most capable" through software means building screening and ranking, which is regulated activity in CSJ's core market and carries real fairness exposure. See §7 and [current-state.md §10](current-state.md#10-new-risk-automated-employment-decision-tools).

### 1.1 What we are building

A platform that helps K–12 schools build their own **Teacher Hiring Machine**, and that makes CSJ a scalable business in the process.

Those are two goals, and they are in productive tension. The first says: give schools durable recruiting capability, not just a place to post jobs. The second says: the platform's value must accrue to CSJ as reusable, compounding assets — data, relationships, and repeatable service delivery — rather than as bespoke effort per client.

The architectural implication of holding both: **the connected dataset is the product.** Job postings, career fairs, and marketing services are how the dataset is acquired and monetized, but the organization/contact/signal graph is the thing that gets more valuable every year. Design decisions that fragment or degrade that graph are expensive even when they look locally convenient.

### 1.2 Principles

1. **Long-term maintainability over short-term convenience.** A solo developer with AI assistance can produce code quickly; the constraint is comprehension, not typing. Optimize for a model that stays legible.
2. **Business concepts and implementation details are distinct.** An Organization is a business concept. A content type is not. Where the framework's vocabulary and the business's vocabulary differ, the business wins.
3. **Normalized relational models with explicit relationships.** Foreign keys, real constraints, no JSON blobs standing in for structure. See [ADR-0001](adr/0001-orchard-core-boundary.md).
4. **Preserve source data and history.** Never overwrite what was asserted; record what changed, when, by whom or by what. Applies to ingested data, enriched data, and merges.
5. **Provenance on every derived fact.** Anything the system inferred, scraped, or generated carries its source, method, confidence, and timestamp — and is stored separately from facts a human asserted.
6. **Explain tradeoffs before proposing implementation.** Decisions get written down as ADRs before code is specified.
7. **Right-size deliberately.** See §6.
8. **Candidates never pay.** The candidate side is acquisition and data. Monetization runs through employers and institutions, always. This is a commercial commitment with architectural reach — see §3.4.

### 1.3 Non-goals

Stating these is as important as stating the goals, because each is a plausible-sounding thing that would sink the project.

- **Not a standalone ATS product.** We do not sell seat-licensed applicant tracking, and we do not build offer management, onboarding, or HRIS. We *do* build applicant workflow depth into the posting experience itself, gated by posting SKU — because the posting is the product. See [commercial-model.md §6](commercial-model.md#conflict-1--the-ats-non-goal-material).
- **Not a general-purpose CMS product.** Orchard Core is infrastructure we consume, not a capability we extend for others.
- **Not microservices.** See §6.
- **No real-time infrastructure of our own.** Nothing in this domain requires sub-second freshness from us. Where features feel real-time (SMS, voice, scheduling), the real-time behavior is delegated to a provider and consumed via queue or webhook.
- **Not multi-region or high-availability at launch.** Single Azure region, standard SLA.

---

## 2. Ubiquitous language

The single cheapest thing in this document and among the most valuable. These terms are used consistently in code, schema, UI, and conversation. Where the business currently uses a term loosely, the precise meaning is fixed here.

| Term | Meaning | Not to be confused with |
|---|---|---|
| **Organization** | Any institution CSJ tracks or transacts with: a school, a network/CMO, an authorizer, a district, a vendor, a partner. The foundational entity. | "Employer" — a *role* an Organization plays, not a type |
| **Organization Type** | What kind of institution: School, Network/CMO, Authorizer, District, Vendor, Partner | Organization *role* |
| **Organization Role** | A relationship an Organization has to CSJ at a point in time: Employer, Client, Prospect, Partner, Sponsor, **Advertiser**, **Vendor**. An Organization may hold several at once. | Organization Type |
| **Campus / Site** | A physical location operated by an Organization. Not necessarily its own Organization. | Organization |
| **Network / CMO** | An Organization that operates or manages other Organizations. Membership is effective-dated — schools join and leave. | Legal parent |
| **Party** | Any actor — a Person or an Organization. The abstraction that lets us relate humans and institutions uniformly. | User |
| **Person** | A human being, tracked once, regardless of how many roles they hold over time. | Candidate, Contact, User |
| **User / Account** | Authentication credentials. Zero or one per Person. A Person exists whether or not they can log in. | Person |
| **Contact** | A Person in a relationship with an Organization (effective-dated, with a title). People change jobs; the relationship ends, the Person doesn't. | Person |
| **Candidate** | A Person in the context of seeking employment. A role, not a separate entity. | Person, Applicant |
| **Applicant** | A Person who has applied to a specific Job Opening. | Candidate |
| **Job Opening** | The business reality: a vacancy, from approval until filled or cancelled. Spans many Publications. The **stable, indexed URL** belongs here. | Publication, Posting |
| **Publication** | One instance of an Opening being displayed — its own start date, duration, **ranking timestamp**, and consumed entitlement. Exactly 30 or 60 days. | Job Opening |
| **Bump** | Republishing an Opening, which resets its ranking timestamp and returns it to the top of the date-sorted board. Implicitly priced at half a 30-day credit; no explicit SKU exists. | Publication |
| ~~Posting~~ | *Retired term.* Ambiguous between Opening and Publication — the ambiguity that hid the modelling error. Use the precise term. | — |
| **Plan** | A subscription tier (Bronze/Silver/Gold/Platinum) granting **posting slots** plus non-consumable benefits. | Product, Subscription |
| **Slot** | One unit of *concurrent* posting capacity under a Plan. Occupied by a Publication for 30 days, then auto-released. Capacity, not currency. Tier size is sized to an organization's **peak** concurrency. | Credit |
| **Credit** | A consumable à la carte purchase entitling one Publication of a given duration. Depletes permanently. | Slot |
| **Posting-month** | One job displayed for one month. The **only** unit in which slots and credits are comparable, and therefore the unit for all utilization and pricing analysis. | Publication |
| **Product** | A sellable thing in the catalog (Plan, Posting SKU, Fair Table, Marketing Package, Search Engagement, Verification). | Plan, Order |
| **Posting Product (SKU)** | The specific posting type purchased — determines duration, price, and included capabilities. Postings are not fungible. | Posting |
| **Entitlement** | A right held by an Organization, consumable within a **scope**. **Four** kinds: renewable capacity (slot), consumable credit, non-consumable benefit, capability flag. See [ADR-0008](adr/0008-products-plans-and-entitlement.md). | Plan, Holder, Consumption Scope |
| **Holder** | The single Organization that owns an entitlement. Not necessarily the Organization that consumes it. | Entitlement, Consumption Scope |
| **Consumption Scope** | Who may draw on an entitlement: `Self`, `SelfAndDescendants` (a network pool — the default for central purchases), or `ExplicitList`. Evaluated against the effective-dated hierarchy at the moment of consumption. | Holder, Network / CMO |
| **Payer** | The Party that pays for an Order. In practice always an Organization — candidates never pay. Not always the beneficiary. **Use for all money questions.** | Beneficiary, Employer |
| **Beneficiary** | The Party the purchase is *for* — a network buying a pooled allowance, or a Person receiving a school-sponsored service. Non-null; defaults to Payer. **Use for everything that isn't a money question.** Note that *per-school* utilization comes from consumption, not from here — a pooled purchase's beneficiary is the network. | Payer |
| **Engagement** | A service delivery relationship — recruiting, executive search, or marketing — with its own lifecycle. | Order |
| **Market** | A **school-sector** and/or geographic scope in which Job Openings and Fairs exist (charter, public, magnet, independent, pre-K). See [ADR-0002](adr/0002-tenancy-and-market-scope.md). | Brand, Tenant, Product Line |
| **Brand** | A public identity bound to one or more hostnames. Presentation only. K12connect is the **platform brand**; Charter School Jobs® is a market-facing brand. | Market |
| **Product Line** | A coherent group of products with its own catalog and delivery model (Job Board, Fairs, CertifiedK12, Services). Not a scope — handled entirely by the Commerce context. | Market, Brand |
| **Signal** | An observed fact about an Organization from an external or behavioral source: a job posted elsewhere, a leadership change, an ad interaction, a site visit. The raw material of marketing intelligence. | Enrichment |
| **Enrichment** | A derived attribute produced from Signals or AI, always carrying provenance. Never overwrites an asserted fact. | Signal, Attribute |

> **Rule:** if a new term is needed, it goes in this table before it goes in code.

---

## 3. Business capabilities

What the business does, independent of how software supports it. Organized by value stream.

### 3.1 Demand side — Employers

| Capability | Notes |
|---|---|
| Employer account management | Self-service org profile, users, branding |
| Annual posting plans | Purchase, entitlement tracking, renewal |
| Individual job postings | À la carte purchase and publication |
| Opening authoring and publication | Create, approve, publish, **re-publish (bump)**, syndicate, expire |
| Applicant delivery | Route applications to the employer's chosen destination |
| Employer marketing services | Sponsored placement, email, LinkedIn advertising |
| Career fair participation | Booth purchase, staffing, lead capture (virtual and in-person) |
| Recruiting and executive search | Delivered as service engagements |
| Employer analytics | Opening performance, **time-to-fill**, applicant flow, benchmarking |

### 3.2 Supply side — Candidates

| Capability | Notes |
|---|---|
| Job discovery and search | Public, no account required |
| Candidate profile | Optional account; resume, credentials, preferences |
| Application submission | The primary conversion event |
| Job alerts and candidate marketing | Email, retargeting |
| Career fair attendance | Registration, scheduling, virtual participation |

### 3.3 CSJ internal — the part usually forgotten

| Capability | Notes |
|---|---|
| **Organization intelligence** | Identity resolution, enrichment, hierarchy maintenance. *Phase 1.* |
| **CRM** | Prospects, contacts, activities, pipeline |
| **Order-to-cash** | Quote, order, invoice, payment, renewal |
| **Service delivery** | Engagement management for search and marketing work |
| **Content and marketing operations** | Site content, campaigns, SEO |
| **Business analytics** | Revenue, retention, funnel, market coverage |

### 3.4 The commercial model

> Derived from source documents and analyzed in full in **[commercial-model.md](commercial-model.md)**. Summarized here; that document governs.

This is the section most often missing from an architecture document, and it is the one that determines whether the model holds up. Revenue arrives in **nine structurally different shapes**, and a single "Order" concept cannot represent all of them well.

| # | Shape | Example | Payer |
|---|---|---|---|
| 1 | **Periodic allowance subscription** | Annual posting plan (monthly credit grant) | Employer |
| 2 | **Transactional** | Single posting credit, priced by duration | Employer |
| 3 | **Event-scoped** | Career fair table, price modified by tier | Employer |
| 4 | **Campaign** | Employer/candidate marketing | Employer |
| 5 | **Service engagement** | Recruiting, executive search | Employer |
| 6 | **Advertising / sponsorship** | Ed schools, PD providers | **Third-party org** |
| 7 | **Metered verification** | Per-query credential verification | Employer |
| 8 | **Sponsored candidate services** | Ed school outsources certification support | **Institution** |
| 9 | **Contract staffing** *(unconfirmed)* | Temp, sub-to-perm | Employer |

> **Standing constraint: candidates never pay.** The candidate side is acquisition and data, never revenue — CV, portfolio, and certification-guidance tools are free, and are measured on candidate acquisition and profile completeness. Where a candidate-benefiting service is monetized, an institution pays (shapes 7 and 8). Changing this requires an ADR. See [commercial-model.md §2.1](commercial-model.md#21-constraint-candidates-never-pay).

**Three structural consequences.**

**a) The payer is not always the beneficiary.** `Order` carries a **Payer Party** and a **Beneficiary Party**. This is already true in the core business today: Platinum's 2,400 postings/year only makes sense for a network buying on behalf of member schools, so "who paid" and "whose postings are these" already differ — and the planned KPI *"average individual-credit spend per school"* cannot be answered without the distinction. Shapes 6–8 extend it. This is why §4.2's Party abstraction exists.

**b) Entitlement has four distinct kinds**, not one:

- **Renewable capacity ("slots")** — what the annual plans actually grant. N *concurrent* slots; a posting occupies one for 30 days, then it auto-releases. Not a depleting balance. The constant question is "can this org publish right now?", which is a **concurrency check**, not a balance lookup.
- **Consumable credit** — à la carte postings. Durable balance, drawn down permanently, SKU'd by duration (30/60-day), sold in quantity bands with volume pricing.
- **Non-consumable benefit** — the career fair discount: a pricing modifier on a *different* product line, valid while the subscription is active. Not a quantity at all.
- **Capability flag** — features unlocked inside a purchased unit (ATS depth within a posting).

An Organization can hold capacity *and* credits at once; both must be resolved to answer whether it may post. Occupancies are materialized records, not computed, so utilization is auditable and policy changes apply forward without rewriting history. See [current-state.md §1](current-state.md#1-the-entitlement-model--my-earlier-reading-was-wrong-twice).

**c) Publications are not fungible.** A Publication references a **PostingProduct** (SKU) determining duration, price, and included capabilities. What a publication *costs* and what it *does* both vary.

**d) Pricing is a schedule, not a number.** À la carte postings use **quantity-break bands** (1/2/3/4/5/10/15/20/25), with the 60-day price exactly **1.5× the 30-day** price at every band — so duration is a multiplier over a base schedule. `Product` therefore owns a price schedule, and `OrderLine` records the resolved unit price at time of sale. Career fair tables add a second mechanism: price modified by the buyer's active entitlements. See [current-state.md §3](current-state.md#3-à-la-carte-structure-and-the-full-price-schedule).

**e) Plan tiers are data, not an enumeration.** Bronze/Silver/Gold/Platinum are **rows** carrying `(slot capacity, price, event discount, term, IsPubliclyListed)`. A negotiated **Custom** plan for a large network is then one more row rather than a code change, and can be hidden from public pricing. Subscriptions reference an immutable **plan version**, so renegotiating a network's terms never rewrites what they were previously sold. Enterprise deals additionally require multi-year terms, invoice/PO billing rather than card, and bundled fair tables. Cheap to build in now, painful to retrofit. See [current-state.md §3.4](current-state.md#34-the-gap-above-platinum--a-custom-tier).

**f) The unit of consumption is the posting-month.** A slot is one job displayed for one month, renewed indefinitely; so is a 30-day credit. Utilization, counterfactual pricing, and tier recommendations must all be computed in posting-months, never in "postings per year" — customers advertise continuously, so distinct-openings counts understate consumption by the average time-to-fill.

**g) Board position is a purchased attribute.** The board is sorted by publication date, so republishing bumps an Opening to the top. The price schedule already implies a **bump is worth exactly half a 30-day credit at every band** — which is why two 30-day credits cost more than one 60-day for the same display duration. Two consequences: there is no explicit bump/refresh SKU although the price is already implied, and plan holders receive one bump per slot per 30-day cycle *included*, which the §3.4 economics undercount. See [current-state.md §3.3](current-state.md#33-the-plan-vs-à-la-carte-economics--corrected).

**The unifying rule:** entitlement resolution is one service, one place. Nothing else in the platform asks "does this org have a plan?" Scattered `if (org.HasPlan)` logic is the standard way a platform like this becomes unmaintainable.

**Preserve what was true at the time.** Order lines carry immutable price-and-terms snapshots; subscription tier is effective-dated history, not a current-value column. Both are required by the planned lifecycle marketing ("what would a plan have saved you last year?") and by the KPI set. Principle #4, applied to commerce.

---

## 4. Core domain model

> Detailed modeling is Phase 1 work and will be specified in ADRs 0003–0007. This section fixes the shape, not the columns.

### 4.1 Organization is foundational — but the entity is the easy part

The hard problems, in order of how much damage they cause if deferred:

**a) What *is* one?** A charter school can be simultaneously a campus, a legal LEA, a member of a CMO, and a line item in a state authorizer's registry. These are four different things that the word "school" covers. We model the legal/operating entity as the Organization and represent campuses, network membership, and authorizer relationships as *effective-dated relationships*, not as attributes.

**b) Hierarchy changes over time.** Schools join and leave networks. Networks merge. A hierarchy modeled as a `ParentId` column loses this history and violates principle #4. Hierarchy is a relationship table with `ValidFrom`/`ValidTo`.

**c) External identity is the whole game.** Marketing intelligence is fundamentally a matching problem: is the school in this LinkedIn ad account the same as the one in the NCES file, the state charter registry, and our CRM? We maintain an **external identifier crosswalk** from day one — NCES ID, state charter ID, EIN, email domain(s), LinkedIn company ID, website — each with its own source and confidence. Without this, every enrichment feature is guesswork.

**d) Organizations have a public face, and it is governed.** Beyond identity, an Organization carries **display name** (distinct from legal name), **logo assets**, **featured flag and sort weight per context**, and **logo usage permission**. Public featuring is gated on an *active* Employer/Client role, never on ever-having-been-a-customer — a former customer shown among fifteen prominent logos is a claim, where the same name buried in a list of 140 was not. See [current-state.md §7.1](current-state.md#71-the-list-is-being-retired--four-things-to-handle-first).

**e) Merges and splits are inevitable.** Two records turn out to be one. A network splits. Merge must preserve both source records and record the merge event; it must not destroy history. This is a Phase 1 requirement, not a later cleanup feature.

### 4.2 Party / Person / Account separation

One human is, over a decade, potentially: a candidate, a hired teacher, then a principal who becomes an employer contact, then a CSJ client, then a fair speaker. If Candidate and Contact are separate root entities each with a glued-on login, this becomes unrecoverable.

```
Party (abstract)
├── Person          — the human, tracked once
└── Organization    — the institution

Account             — credentials; 0..1 per Person; may not exist at all
OrganizationContact — Person ↔ Organization, effective-dated, with title
Candidacy           — Person's job-seeking profile and preferences
Application         — Person → JobOpening
```

A Person exists without an Account. A Person may hold many roles simultaneously and historically. Roles are relationships, never subclasses.

### 4.3 Job Opening and Publication are separate entities

*Added 2026-08-06. This corrects an error: earlier drafts modelled "Posting" as one entity.*

The job board is sorted by posting date, and republishing an Opening bumps it to the top. That mechanic is only representable if the vacancy and its appearances on the board are distinct:

```
JobOpening            — the vacancy. Created → Filled | Cancelled.
  │                     Owns the stable, indexed URL. Spans months.
  ├── Publication      — 30 or 60 days. Has its own RankedAt timestamp.
  ├── Publication      — a repost: new RankedAt, same Opening, same URL.
  └── Publication
        └── consumes one Slot occupancy or one Credit
```

**Three timestamps that must not be collapsed into one:**

| Field | Meaning | Used for |
|---|---|---|
| `Opening.OpenedAt` | When the vacancy became real | Time-to-fill, "how long has this been open?" |
| `Publication.PublishedAt` | When this appearance began | Display window, entitlement accounting |
| `Publication.RankedAt` | The board's sort key | Search ordering — **the monetized attribute** |

They coincide on a first publication and diverge on every repost. Conflating them is how a system loses the ability to answer its most valuable question.

**What this unlocks, none of it currently measurable:** time-to-fill (the headline metric for a platform selling *"cost-effectively staff my school"*), honest posting-month accounting per role, fill rate and repost rate, and suppressing duplicate-looking reposts for candidates who already applied.

**URL identity is settled.** The Opening owns one permanent canonical URL (`/jobs/{publicCode}/{slug}`); publications mutate what it emits, never mint new URLs. Lapsed Openings stay 200 and indexed so authority survives to the next bump; `JobPosting` structured data is emitted only while a publication is active. Organization is deliberately absent from the path, because Organizations merge and rebrand (§4.1) and that must not be an SEO event. See **[ADR-0013](adr/0013-opening-identity-and-canonical-urls.md)**.

**Publication snapshots are a legal record, not just a model convenience.** NY Labor Law §194-B requires a good-faith wage range and written job description in postings for New York roles, *and* requires employers to retain a history of both. Each Publication therefore stores an immutable snapshot of title, description, and compensation range as advertised. → ADR-0014.

**Ranking position is a commercial attribute.** `RankedAt` is purchasable, so if relevance ranking is ever added the bump becomes a paid ranking signal — a question adjacent to §7's AEDT concerns.

### 4.4 Signals, enrichment, and provenance

This pattern underpins Employer Marketing Intelligence and is what makes AI enrichment safe:

```
RawObservation   — immutable. The payload exactly as received, with source,
                   fetched-at, and a content hash. Never edited, never deleted.
        │
        ▼
Signal           — normalized interpretation of an observation, linked back to it
        │
        ▼
EnrichedAttribute — a derived claim about an Organization or Person, carrying:
                    value, source, method (rule | model | human), model version,
                    confidence, observed-at, and review status
        │
        ▼
AssertedFact     — what a human confirmed. Separate storage. Always wins.
```

**Three rules that follow:**

1. AI output never writes silently into a canonical field. It produces an `EnrichedAttribute` awaiting review or auto-accept-above-threshold.
2. Every derived value can be traced to the raw payload it came from.
3. Re-running enrichment with a better model is safe, because nothing was overwritten.

This makes §10 (AI strategy) largely unnecessary as prose — the constraints are structural.

---

## 5. Bounded contexts

At CSJ's scale these are **module boundaries inside one deployable**, enforced by assembly reference rules and SQL Server schema ownership — not services, not separate databases. See [ADR-0001](adr/0001-orchard-core-boundary.md).

| Context | Schema | Owns | Depends on |
|---|---|---|---|
| **Organizations** | `Org` | Organization, hierarchy, external identifiers, merges | — |
| **Parties** | `Party` | Person, Account link, contacts | Organizations |
| **Intelligence** | `Intel` | Observations, signals, enrichment, provenance | Organizations, Parties |
| **Commerce** | `Commerce` | Products, orders, entitlements, invoicing | Organizations |
| **Hiring** | `Hiring` | Job Openings, publications, applications, candidacy | Organizations, Parties, Commerce |
| **Events** | `Events` | Career fairs, booths, registrations | Organizations, Parties, Commerce |
| **Engagements** | `Svc` | Recruiting/search/marketing service delivery | Organizations, Parties, Commerce |
| **CRM** | `Crm` | Prospects, activities, pipeline | Organizations, Parties |
| **Content** | *(YesSql)* | Marketing site, media, navigation | — (reads domain via services only) |

**Dependency rules**

- Organizations depends on nothing. It is the root. This is what "Organizations is foundational" means concretely.
- Dependencies point downward in the table. No cycles. If a cycle appears, the boundary is wrong.
- Cross-context access goes through application services, never cross-schema joins.
- Content may read domain data. Domain never reads content.

---

## 6. Right-sizing — the scale we are actually building for

Written down so that no future decision quietly assumes otherwise.

> **Footprint correction (2026-08-06).** CSJ is not Metro-NY-only. The live customer list (~140 organizations) includes NJ, CT, PA, DC, NC, MI, TX, CA and LA — national CMOs such as KIPP, Aspire, Uplift, and National Heritage Academies already buy. Metro NY is the **core**, not the boundary. This does not change the magnitudes below, but it does make geographic scope a *present* condition rather than a future scenario. See [current-state.md §6](current-state.md#6-geographic-reality-contradicts-the-metro-new-york-framing).

| Dimension | Realistic magnitude |
|---|---|
| Organizations tracked | Hundreds now; low thousands at K–12 scale |
| Active postings | Hundreds concurrent |
| Publications per year | Low thousands |
| Candidates | Tens of thousands |
| Applications per year | Tens of thousands |
| Concurrent users | Dozens, peaking in hiring season |
| Development team | One, with AI assistance |

**Therefore, explicitly:**

- **Modular monolith.** One deployable, one database. No microservices, ever, for this scale.
- **No message bus** until a specific requirement demands one. In-process events and a simple background job queue cover everything foreseeable.
- **No CQRS, no event sourcing** as global patterns. The provenance model in §4.4 gives us the history we actually need, at far lower cost.
- **No separate analytical database** initially. SQL Server with indexed views and read-optimized tables is sufficient at these volumes. Revisit only when a query is genuinely slow, measured.
- **No caching layer** beyond ASP.NET Core in-memory and Cloudflare edge caching for public pages.

The hard constraint is not throughput. It is **one person's ability to hold the system in their head.** Every piece of infrastructure is a claim on that budget.

---

## 7. Cross-cutting concerns

Summarized here; detailed in ADRs.

**Identity and authorization** — Orchard Core owns authentication and the user store. The domain owns Person and organization-scoped permissions. Employers need to delegate access to colleagues without CSJ staff intervention; that is an org-scoped role model, not a global role list. → ADR-0003, ADR-0009

**Integration** — LinkedIn advertising, email/marketing automation, payments, job board syndication, public education datasets (NCES, state registries). All follow §4.4: ingest to `RawObservation` first, normalize second. No integration writes directly to canonical domain fields. → ADRs as each is built

**AI** — Constrained structurally by §4.4 rather than by policy prose. Near-term applications: organization identity resolution and deduplication, contact and hierarchy enrichment, posting quality assistance for employers, candidate-posting matching. Every one produces reviewable claims, never silent writes. → ADR-0011

**Automated employment decisions (regulatory)** — Candidate scoring, ranking, or screening is an *automated employment decision tool* under NYC Local Law 144, in force since 2023 and squarely applicable to CSJ's core market. The obligations — independent annual bias audit, public posting of results, 10 business days' candidate notice — **fall on the employer, not the vendor**. Shipping ranking without supplying audit artifacts and notice mechanisms would put CSJ's own customers out of compliance. Three design requirements follow: scoring must be reproducible and inspectable per decision (the §4.4 provenance chain, applied to matching); **ranking must be architecturally separable from search and filtering**; candidate notice and audit surfaces are product features, not afterthoughts. Not legal advice — counsel should confirm scope before build. → ADR-0011, brought forward. See [current-state.md §10](current-state.md#10-new-risk-automated-employment-decision-tools).

**Observability** — Application Insights. Structured logging with correlation IDs. Given a solo operator, alerting matters more than dashboards.

---

## 8. Coexistence and cutover

*Deliberately not called "migration."* The domain model is greenfield; the risk is not data conversion.

The real exposure:

1. **SEO equity.** The existing Orchard 1.10.3 site has indexed job and employer URLs. Losing that ranking is a direct revenue hit and is not recoverable by launching a better site. URL continuity is a launch requirement, not a nice-to-have. The new scheme is settled in [ADR-0013](adr/0013-opening-identity-and-canonical-urls.md); ADR-0010 covers mapping the legacy estate onto it. **If the legacy system minted a new URL per repost, the principal cutover work item is consolidating those clusters** — identify legacy URLs representing one vacancy, pick the canonical target from Search Console link data, 301 the rest, and preserve the mapping as data. Establishing that behaviour is the first thing to do. → ADR-0010
2. **In-flight commercial obligations.** Customers hold annual plans with remaining posting balances. The new entitlement model must be able to represent an obligation that was sold under the old system's terms.
3. **Live postings and inbound applications.** Applications must not be lost during cutover. This constrains the sequencing more than anything else.
4. **Historical data.** Preserve, don't discard — but import into the new model as *observations with provenance* (source: legacy system), not as canonical asserted facts. This lets us clean data over time without pretending the legacy values were verified.

**Approach:** strangler fig. Cloudflare routes paths to old or new by URL pattern, moving progressively. This requires that the new platform be able to serve a subset of paths before it is complete — a real constraint on how modules are built, worth knowing now.

---

## 9. Roadmap

Each phase must ship something a user or the business can actually use. A domain model with no consumer is almost always wrong, and the way you find out is by shipping.

### Phase 0 — Foundations (current)

Blueprint and Phase 1 ADRs. Solution structure. CI with the `CSJ.Domain.* must not reference OrchardCore.*` guardrail. Azure environment.

### Phase 1 — Organizations + one thin revenue slice

Organizations, hierarchy, external identifier crosswalk, merge/split, Party/Person, and the provenance foundation — **paired with** employer account self-service and one working commercial path (plan purchase → entitlement → posting). The commercial slice exists specifically to falsify the Organizations model against real use before eight modules sit on top of it.

*Exit criterion: a real employer buys a plan, posts a job, and receives an application.*

### Phase 2 — Employer Marketing Intelligence

Signal ingestion (LinkedIn, email, web, public datasets). Enrichment pipeline with review workflow. CRM. Organization intelligence dashboard. This is where the Phase 1 investment in identity resolution pays off.

### Phase 3 — Hiring depth and Events

Full posting lifecycle, candidate profiles, application management, career fairs.

### Phase 4 — Service delivery and analytics

Recruiting/search engagements, marketing campaigns, employer-facing analytics.

### Phase 5 — K–12 expansion

Additional markets. Whether this means new `Market` records or something larger depends on [ADR-0002](adr/0002-tenancy-and-market-scope.md).

---

## 10. Open questions

Blocking or near-blocking. Tracked here until resolved into an ADR.

1. ~~**K12connect's relationship to CSJ**~~ — **RESOLVED 2026-08-06.** K12connect is the platform/corporate brand; Charter School Jobs® is a market-facing brand; CertifiedK12 is a distinct product line sharing the Organization and Person graph. Expansion is by school **sector** as much as geography. See [commercial-model.md §7](commercial-model.md#7-answering-blueprint-open-question-1--what-is-k12connect).
2. ~~**Do plan credits reset monthly or pool annually?**~~ — **DISSOLVED 2026-08-06.** Plans grant renewable *capacity* (slots with 30-day occupancy), not consumable credits. Nothing expires, so nothing rolls over. See [current-state.md §1](current-state.md#1-the-entitlement-model--my-earlier-reading-was-wrong-twice).
3. **Do networks/CMOs buy centrally for member schools, and is capacity pooled or allocated?** One central pool is simplest but gives no per-school accountability and lets one school starve the others; per-school allocation from a parent entitlement needs hierarchy-aware delegation but is what the *"average spend per school"* KPI actually requires. A **Custom tier** for a network the size of Success Academy makes this urgent rather than theoretical. Blocks both the entitlement model and ADR-0004. See [current-state.md §3.4](current-state.md#34-the-gap-above-platinum--a-custom-tier).
4. **How do plan slots and à la carte credits interact?** Which is consumed first? Does a 60-day credit occupy a slot for 60 days, or are the two systems independent? **Blocking the entitlement model.**
5. **Is a Custom/Enterprise tier in scope for the rebuild?** At 200 concurrent roles a plan already saves 96% against à la carte, so Platinum is substantially underpriced at the top of its range and prices a 60-slot and a 200-slot network identically. Commercial decision; the four modelling consequences are in [current-state.md §3.4](current-state.md#34-the-gap-above-platinum--a-custom-tier) and consequence (e) of §3.4 above should be built regardless.
6. **What is the subscription term anchored to** — calendar year, school year, or purchase anniversary? Drives renewal clustering and proration.
7. **What defines an Organization in the charter context** — legal LEA, operating entity, or campus? Blocks ADR-0004. The live customer list shows the ambiguity is already in the data: Great Oaks, Lighthouse Academies, and Citizens of the World each appear as both network and campus.
8. **What happens when all slots are occupied** — blocked, queued, or upsold to à la carte? Likely a real revenue line, currently undocumented.
9. **Does the legacy system mint a new URL per repost, and what is the job URL pattern?** Almost certainly the highest-volume indexed URLs on the site. Determines whether cutover includes a large repost-cluster consolidation. Needs the legacy URL pattern, a sitemap, and a Search Console export. **Blocks ADR-0010 scoping** — the new-system decision is already made in [ADR-0013](adr/0013-opening-identity-and-canonical-urls.md).
10. **Is AI candidate ranking in scope for the rebuild, and has counsel reviewed AEDT exposure?** See §7.
11. **Should internal postings** (K12 Staffing, K12connect Search) **be flagged and excluded** from commercial reporting?
12. **Which authoritative external datasets are available and licensable** for K–12 school data? Determines how much of the identifier crosswalk can be seeded vs. hand-built.
13. **Application delivery** — email, in-platform, or into the employer's own ATS? Determines how deep Phase 1's hiring slice must go.
14. **Payments** — what is used today, and is there a constraint on changing it?

**Resolved since v0.1:** ~~temp/contract staffing real?~~ (yes) · ~~do credits roll over?~~ (dissolved — slots are capacity) · ~~fair discount tiered or flat?~~ (**flat 20%**, confirmed by live site *and* internal price sheet; the Conversion Strategy is in error) · ~~à la carte price schedule?~~ (**retrieved** — 30-day $100 base falling to $50 at qty 25; 60-day is exactly 1.5× at every band).

Full reasoning: [commercial-model.md §8](commercial-model.md#8-open-questions-for-kent) and [current-state.md §11](current-state.md#11-open-questions-raised-or-changed).
