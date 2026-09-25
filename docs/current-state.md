# Current State — Live Site Analysis

**Status:** Analysis, v0.1
**Date:** 2026-08-06
**Sources:**

- `charterschooljobs.com`, observed 2026-08-06 (Orchard 1.10.3, `BoomerangTemplate` theme). Pages reviewed: home, Annual Posting Plans, Buy Posting Credits, plus common navigation/footer chrome.
- *Charter School Jobs Price Sheet* (Google Sheet, modified 2026-08-06) — the authoritative price schedule, including the à la carte prices the live page renders via JavaScript.
- *K12connect Mission Statement (2022)* (Google Doc).

This document records **what the live system actually does**. Where it contradicts a strategy document, the live system is treated as the fact and the contradiction is flagged — not silently reconciled.

---

## 1. The entitlement model — my earlier reading was wrong twice

**[commercial-model.md §1.2](commercial-model.md) framed the question as "monthly allowance vs. annual pool." It is neither.** The plans page states the mechanism directly:

> *"Our annual plans include a fixed number of job posting **'slots'**... **30 days after a slot is used, it becomes available for a fresh new job posting** at the top of the (chronologically sorted) job listings... That means no more worrying about buying additional credits when your postings expire, for a full year!"*

This is a **renewable capacity lease**, not a consumable balance:

| | Consumable credit | **Slot (actual model)** |
|---|---|---|
| Nature | Stock, drawn down | **Capacity, occupied then released** |
| Depletion | Permanent | **Temporary — 30-day occupancy** |
| Limit | Total count | **Concurrency** |
| "Rollover" | Meaningful question | **Meaningless — slots don't expire** |

A Bronze customer holds **5 concurrent slots**, each recycling every 30 days. "5 job posting credits per Month" on the pricing card is *derived throughput* (5 slots × 30-day cycle ≈ 5/month), not the entitlement itself.

**Consequences:**

1. **My open question "do credits roll over?" is dissolved, not answered.** Slots don't expire, so there is nothing to roll over. Withdrawn.
2. **The binding constraint is concurrency, not a monthly cap** — and that is by design. *(Revised 2026-08-06: an earlier version called this "hostile to seasonal hiring." It isn't. The **tier ladder is the answer to seasonality** — an organization buys the tier that covers its peak, which is why Platinum's 200 slots exist and why both current Platinum customers routinely exceed 50 concurrent postings. See §3.3.)* The residual trade is that sizing to peak means carrying idle capacity off-season — normal for a reserved-capacity product, but worth watching for tier-downgrades clustering after hiring season.
3. **Two genuinely different currencies coexist.** Plans grant *slots* (capacity). À la carte grants *credits* (consumable, 30- or 60-day). These are not interchangeable, and the platform must answer "can this org post right now?" against both. **Consumption order decided 2026-08-06** — *perishability first, then proximity*: slots before credits, own before an in-scope ancestor's. See [ADR-0008 §5](adr/0008-products-plans-and-entitlement.md).
4. **The site uses "slots" and "credits" interchangeably on the same page** — headline copy says slots, pricing bullets say credits. This is exactly the ambiguity the blueprint glossary exists to eliminate, found in the wild on the highest-value page on the site.

### Corrected entitlement kinds — four, not three

| Kind | Example | Mechanic |
|---|---|---|
| **Renewable capacity (slot lease)** | Annual plan | N concurrent slots, 30-day occupancy, auto-release |
| **Consumable credit** | À la carte posting | Durable balance, drawn down, SKU'd by duration |
| **Non-consumable benefit** | Career fair discount | Pricing modifier on another product line |
| **Capability flag** | ATS depth within a posting | Feature gate tied to posting SKU |

---

## 2. Contradiction: the career fair discount is not tiered

> ✅ **RESOLVED — see §3.2.** The internal price sheet independently confirms flat 20% on all four tiers. The Conversion Strategy's tiered ladder is an error in that document. The commercial consequence below still stands.

**The Conversion Strategy states** the plans "bundle a career fair table discount (5–20%)" and its comparison table assigns Bronze 5%, Silver 10%, Gold 15%, Platinum 20%.

**The live site states, four times, once per tier:** *"20% discount on Charter School Jobs® career fair tables."* Plus a banner above all four: ***"ALL plans come with a 20% discount on employer tables at our career fair events."***

Every tier gets 20%. There is no discount ladder.

**Why this matters commercially.** The Conversion Strategy's central recommendation — §4, the "connective tissue" between fairs and plans — rests substantially on the discount stack as an upgrade lever ("plan holders save up to 20%", tier-by-tier savings quantified in dollars). As configured, **the fair discount provides zero incentive to upgrade tiers.** It's a reason to hold *a* plan, not a better one.

**This needs resolving before either is built on:**

- If the tiered ladder is the intent, the site is wrong and is currently giving Bronze customers a 15-point overage.
- If flat 20% is the intent, the Conversion Strategy's upgrade argument needs rebuilding on a different lever (slot count is the obvious one).

Either way, **the architecture is unaffected** — a non-consumable benefit entitlement expresses flat or tiered equally well. This is a business decision, flagged because two canonical documents disagree. **[CONFIRM]**

---

## 3. À la carte structure and the full price schedule

**Source added 2026-08-06:** *Charter School Jobs Price Sheet* (Google Sheet, modified 2026-08-06). This supplies the à la carte pricing the Conversion Strategy could not retrieve, and it resolves §2.

### 3.1 The schedule

`/JobPostingProducts/BuyPostingCredits` offers posting type (30-Day / 60-Day) × quantity band, with per-unit price falling by band:

| Qty band | 1 | 2 | 3 | 4 | 5 | 10 | 15 | 20 | 25 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| **30-day** | $100 | $95 | $90 | $85 | $80 | $70 | $65 | $60 | $50 |
| **60-day** | $150 | $142.50 | $135 | $127.50 | $120 | $105 | $97.50 | $90 | $75 |

**The 60-day price is exactly 1.5× the 30-day price at every band** — verified across all nine. Duration is therefore a **multiplier on a base price**, not an independent price list. That's a much cleaner rule to model than two parallel schedules, and it generalizes if other durations are ever added.

*Note:* the sheet labels the mechanism "Incremental Discount: 5.00%", but the actual band prices are hand-set and don't follow a 5% rule (1→2 is −5%, 2→3 is −5.3%, 20→25 is −16.7%). Model it as a **stored band table**, not a computed discount. Also, the sheet's displayed 4-qty and 15-qty unit prices ($128, $98) are rounded; the subtotals imply $127.50 and $97.50. Store the exact values.

Two mechanisms confirmed for the model: **Posting SKU by duration** (blueprint §3.4c) and **quantity-break pricing** — so `Product` needs a *price schedule*, not a price.

### 3.2 §2 is resolved — the discount is flat 20%

The price sheet's Annual Plan table reads:

| Level | Price | Postings/month | Event Discount |
|---|---:|---:|---:|
| Bronze | $800 | 5 | **20%** |
| Silver | $1,500 | 15 | **20%** |
| Gold | $2,500 | 50 | **20%** |
| Platinum | $4,000 | 200 | **20%** |

Two independent sources — the live site and the internal price sheet — agree on **flat 20% across all tiers**. The Conversion Strategy's tiered 5/10/15/20% ladder is an **error in that document**, not a live configuration.

**Downgraded from blocking to resolved**, with one consequence that stands: the Conversion Strategy's §4 "connective tissue" recommendation leans on the discount ladder as an upgrade lever, and that lever does not exist. The fair discount is a reason to hold *a* plan, not a better one. If tier upgrades need a lever, slot count is the only one currently available.

Also note the price sheet labels slots as **"Postings/month"** — a third vocabulary for the same thing, alongside "slots" and "credits" on the live site. Same object, three names, three documents.

### 3.3 The plan-vs-à-la-carte economics — corrected

> ⚠️ **RETRACTED AND REPLACED 2026-08-06**, after Kent's correction. An earlier version of this section computed break-even in **postings per year** and concluded that à la carte beat Bronze below ~12 postings, and that Gold and Platinum were barely differentiated. **Both conclusions were wrong, and they were wrong for the same reason:** the analysis used *annual throughput* as the unit while the product is sold in *concurrent capacity*. §1 of this document identifies the slot model correctly and then the pricing analysis silently reverted to a consumable-credit mental model. The corrected analysis is below. The retracted reasoning is not reproduced because, unlike the §1.2 correction in `commercial-model.md`, none of it survives.

#### The right unit is the posting-month

A slot is not "a posting." A slot is **one job displayed for one month**, renewed indefinitely. So is a 30-day credit. That is the only unit in which plans and à la carte are comparable.

This matters because CSJ's customers advertise **continuously** — national teacher shortage, high turnover across instructional *and* non-instructional staff. A role isn't posted once; it stays up until filled, and is often re-opened. Counting "distinct openings per year" understates consumption by whatever the average time-to-fill is.

#### Corrected comparison

For an organization keeping **P roles open year-round** (à la carte priced with 60-day credits, the cheaper option — see below):

| Roles open year-round | Cheapest plan | Plan cost | À la carte | **Plan saves** | Plan $/posting-month |
|---:|---|---:|---:|---:|---:|
| 1 | Bronze | $800 | $720 | −11% | $66.67 |
| 2 | Bronze | $800 | $1,260 | **37%** | $33.33 |
| 3 | Bronze | $800 | $1,755 | **54%** | $22.22 |
| 5 | Bronze | $800 | $2,250 | **64%** | $13.33 |
| 10 | Silver | $1,500 | $4,500 | **67%** | $12.50 |
| 15 | Silver | $1,500 | $6,750 | **78%** | $8.33 |
| 25 | Gold | $2,500 | $11,250 | **78%** | $8.33 |
| 50 | Gold | $2,500 | $22,500 | **89%** | $4.17 |
| 100 | Platinum | $4,000 | $45,000 | **91%** | $3.33 |
| 200 | Platinum | $4,000 | $90,000 | **96%** | $1.67 |

*(Above 25 units the qty-25 band is extrapolated; the published schedule stops there, so real à la carte cost is likely higher and the plan advantage larger.)*

**Break-even is one continuously-open role.** Any organization with two or more roles open at once is already better off on a plan, and the advantage widens fast. The earlier claim that the funnel mis-serves low-volume schools was wrong — pointing every employer at Annual Posting Plans is the *correct* default. The only buyer à la carte genuinely suits is a school filling a single role occasionally, which is the one-off case the product already exists for.

#### The marketing figures are not aspirational

Bronze at full utilization is $800 ÷ (5 slots × 12 months) = **$13.33 per posting-month** — which is the published $13.31, to rounding. Under year-round advertising, full slot utilization is the *normal* case, not a theoretical ceiling.

**This retracts [commercial-model.md §1.3](commercial-model.md).** That section argued the per-posting figures assumed unrealistic 100% utilization and would be contradicted by an honest "what would you have saved" feature. The opposite is true: built correctly in posting-months, that feature will *confirm* the marketing and produce larger savings numbers than the pricing table claims, because it compares against à la carte rates the table never shows.

#### The 30/60-day differential prices *visibility*, not duration

> **Corrected 2026-08-06.** An earlier version called the 30-vs-60-day gap an unintentional pricing anomaly and concluded nobody should buy 30-day credits. That was wrong, and it missed a product mechanic that turns out to be architecturally significant.

The job board is **sorted by posting date**. Two 30-day credits are not merely two months of display — the second posting **resets the date and returns the job to the top of the board**. One 60-day credit buys the same two months but sinks steadily for all of them.

So the 30-day SKU isn't the worse deal. It's the same display duration **plus a bump**, and the price gap is what the bump costs.

**The pricing is internally coherent, and exactly so.** Because 60-day is priced at 1.5× the 30-day at every band, the implied cost of one bump is **precisely half a 30-day credit — at every band, without exception**:

| Qty band | 1 | 2 | 3 | 4 | 5 | 10 | 15 | 20 | 25 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 2 × 30-day | $200 | $190 | $180 | $170 | $160 | $140 | $130 | $120 | $100 |
| 1 × 60-day | $150 | $142.50 | $135 | $127.50 | $120 | $105 | $97.50 | $90 | $75 |
| **Implied bump** | **$50** | $47.50 | $45 | $42.50 | $40 | $35 | $32.50 | $30 | **$25** |

That the ratio holds to the cent across nine bands means the 1.5× multiplier is doing deliberate work, not producing an artifact.

**Three consequences that matter.**

**1. Position on the board is a monetizable attribute, and it's currently unpriced.** There is no "bump" or "refresh" SKU. Visibility can only be bought by consuming a whole extra credit, which is a blunt instrument — a bump is worth ~$50 and a credit costs $100. An explicit refresh product is the obvious gap, and the price is already implied by the existing schedule.

**2. Plan holders get bumps automatically, which makes plans more valuable than §3.3 shows.** Per §1, a slot releases after 30 days and the next posting goes in *"at the top of the (chronologically sorted) job listings."* So a Bronze customer receives 5 slots × 12 cycles = **60 bumps a year, included**. The plan-vs-à-la-carte comparison in §3.3 counted only display duration and therefore *understates* the plan advantage. At a $50 implied bump value, Bronze's included bumps alone are notionally worth $3,000 against an $800 price. This is a stronger sales argument than any in the Conversion Strategy, and nobody is making it.

**3. Sort order is the entire ranking algorithm today.** Recency is the only signal. That's simple and fair in an obvious way, but worth naming: if relevance or quality ranking is ever introduced, the paid bump becomes a paid *ranking signal*, which is a different product and a different conversation — one in tension with the mission's emphasis on match quality (blueprint §1.0), and adjacent to the AEDT concerns in §10. Not a problem now. Worth deciding deliberately rather than drifting into.

#### Why the tier ladder is coherent — corrected

**Slots are peak concurrency, not annual volume.** The ladder is sized to what an organization needs *visible simultaneously at peak*, and per Kent both current Platinum customers routinely exceed 50 concurrent postings. Gold's 50-slot ceiling is a hard capability boundary for them, not a marginal price difference — Gold cannot serve a network that needs 120 roles up in March.

This also resolves, mostly, the seasonality concern raised in §1: **the tier ladder is the answer to seasonality.** You buy the tier that covers your peak and carry idle capacity off-season, exactly as with any reserved-capacity product.

Two things remain true and worth noting without overstating them:

1. Sizing to peak means paying for idle capacity much of the year, which is a normal and accepted trade — but it does create a real downgrade temptation at renewal, and a possible future product (short-term burst capacity) if churn analysis ever shows tier-downgrades clustering post-season.
2. The tier ladder tops out at 200. A network materially larger than that has no product. See §3.4.

*Minor:* the savings confirmation on the live page renders as *"You save&nbsp; on this purchase"* — an empty value in the checkout path.

#### The bump mechanic breaks my Posting model

This is the most consequential thing in this document, and it comes out of a pricing detail.

If reposting bumps a job to the top, then **a repost is a new publication of the same vacancy** — and the platform has been modeling one entity where there are two:

| Concept | What it is | Lifecycle |
|---|---|---|
| **Job Opening** | The business reality: a vacancy that exists from the moment it's approved until it's filled or cancelled | Weeks to months; spans many publications |
| **Publication** | One instance of that Opening being displayed: its own start date, duration, **ranking timestamp**, and consumed entitlement | Exactly 30 or 60 days |

Blueprint §2 lists "Position, Requisition" under *not to be confused with* for **Posting** — but then models Posting as a single entity. That was an error, and the bump mechanic exposes it: you cannot represent "this job was posted, expired, and reposted twice" without the split, and that is the *normal* case for a hard-to-fill role in a teacher shortage.

**What the split unlocks — none of which is currently measurable:**

- **Time-to-fill.** Opening-created → Opening-filled, spanning publications. Today only publication windows exist, so the headline recruiting metric can't be computed. For a platform whose pitch is *"cost-effectively staff my school"*, this is the number customers most want and CSJ can't currently show them.
- **Honest posting-month accounting.** Total consumption for a role = sum of its publications' durations.
- **Real fill-rate and re-post-rate.** How many roles go unfilled after three publications? That's both a product-quality signal and a consulting/recruiting-services lead.
- **Renewal and bump behaviour** as first-class events rather than inferred from duplicate rows.
- **Deduplication of the candidate experience.** Today a repost presumably looks like a new job. A candidate who already applied should see that.

**SEO consequence — feeds ADR-0010.** The Opening should own the **stable, indexed URL**; publications change its `datePosted`. If the current system instead mints a new URL per repost, then every hard-to-fill role has been fragmenting its own link equity and generating near-duplicate content for years — which would be both a live SEO problem and a cutover trap, since consolidating those URLs is a redirect-mapping exercise that has to be planned, not discovered. **Confirming the current repost URL behaviour is now a prerequisite for ADR-0010**, alongside the posting URL pattern.

### 3.4 The gap above Platinum — a Custom tier

Kent raises a Custom plan for very large networks, citing Success Academy (a former customer, recently re-engaged through a virtual career fair). The economics support it plainly: at 200 concurrent roles the plan already delivers 96% savings against à la carte, meaning **Platinum is dramatically underpriced for organizations at the top of its range**, and priced identically for an organization needing 60 slots and one needing 200.

This is a pricing decision, not an architectural one — but it imposes four requirements on the model, and they are cheap now and expensive later.

**1. Plan tiers must be data, not an enumeration.** If Bronze/Silver/Gold/Platinum are rows in a table carrying `(slot capacity, price, event discount, term)`, then a Custom plan for one network is one more row. If they are a C# enum or hard-coded configuration, every custom deal becomes a code change. **This is the single most important consequence** — it costs nothing to get right at the start and is painful to retrofit.

**2. Tiers need a visibility flag.** `IsPubliclyListed` — a negotiated plan must not appear on the public pricing page.

**3. Subscriptions reference an immutable plan *version*.** Renegotiating a network's terms must not silently rewrite what they were previously sold. Same principle as order-line price snapshots (blueprint §3.4): preserve what was true at the time.

**4. Enterprise deals bring their own commercial shapes.** Multi-year terms, invoicing and PO numbers rather than card payment, bundled fair tables, and custom event discounts. None of these are exotic, but none are expressible in a model that assumes a public tier bought with a credit card.

**The genuinely interesting question a Custom tier forces** — and it is the same one as blueprint open question #3 — is **slot allocation within a network.** Success Academy operates dozens of schools. Does a 300-slot custom plan mean:

- **One central pool**, drawn by any member school? Simple; no per-school accountability; one school can starve the others.
- **Per-school allocation** from a parent entitlement? Requires hierarchy-aware entitlements and a delegation mechanism, but gives per-school reporting — which is what the Conversion Strategy's *"average spend per school"* KPI actually needs.
- **Hybrid** — a guaranteed floor per school plus a shared overflow pool.

This is a real modeling decision with no default, it lands squarely on the Phase 1 organization-hierarchy work, and a Custom tier makes it urgent rather than theoretical. **[CONFIRM]** — it should be settled with the first enterprise customer in the room, not designed speculatively.

**One CRM note:** that Success Academy is a *former* customer re-engaged through a career fair is exactly the relationship arc the platform must represent — Organization Role as effective-dated history (Customer → Lapsed → Prospect → re-engaged via Event), not a current-status column. The largest deal in the pipeline being a win-back is a strong argument for that design.

---

## 4. Corporate structure — confirmed from the footer

> **"Charter School Jobs® is a registered trademark of K12connect, Inc. K12connect, Inc. ©2004–2026."**

This settles [commercial-model.md §7](commercial-model.md#7-answering-blueprint-open-question-1--what-is-k12connect) from primary evidence rather than inference. **K12connect, Inc. is the legal entity; Charter School Jobs® is a trademark it owns.** The three-layer structure (platform → market-facing brand → product line) is the real corporate structure, not an architectural convenience.

Also note the logo `alt` text reads *"Charter School Jobs, Inc."* while the footer reads *"K12connect, Inc."* — a legacy inconsistency, and a small confirmation that brand/entity separation has never been modeled explicitly.

---

## 5. Two business lines confirmed — from the customer list

The homepage customer wall includes:

- **"K12 Staffing"** → `k12connect.com`
- **"K12connect Search"** → `k12connect.com`

CSJ's own service lines appear as customers of its own job board. This **answers blueprint open question #6**: staffing and search are real, operating business lines, not merely domain registrations. It also explains `charterschooltemps` and `sub2perm`.

**Architecturally this is the most interesting thing on the page.** CSJ is *its own employer customer* — it posts jobs on its own board to source candidates for its staffing and search engagements. That means:

- An Organization can hold role = Employer **and** be internal to K12connect. The role model handles this; a boolean `IsCustomer` flag would not.
- Internal postings must be distinguishable from customer postings for revenue reporting, or staffing-line postings will pollute every commercial metric.
- Placement economics (staffing/search) attach to the *same* Person and Posting entities as the marketplace. This is an argument for the shared graph, not against it.

---

## 6. Geographic reality contradicts the "Metro New York" framing

The project brief describes CSJ as serving "primarily the Metro New York region." The customer list (~140 organizations) is materially broader:

| Region | Examples |
|---|---|
| **NY** (core) | Success Academy, Uncommon, KIPP NYC, Harlem Village, Ascend, Democracy Prep, Explore, Public Prep |
| **NJ** | KIPP New Jersey, Marion P. Thomas (Newark), People's Prep, BelovED (Jersey City), Hoboken Charter, iLearn, LEAD |
| **CT** | Achievement First, Jumoke Academy, Capital Preparatory |
| **PA** | Propel Schools, Scholar Academies |
| **DC / MD** | DC Prep |
| **TX** | KIPP Houston, Uplift Education |
| **CA** | Aspire, Alliance College-Ready, Camino Nuevo, Amethod, iLEAD, Options for Youth |
| **Other** | National Heritage Academies (MI), Pine Lake Prep (NC), The NET (New Orleans) |

**Metro NY is the core, not the boundary.** National CMOs already buy.

**Impact on [ADR-0002](adr/0002-tenancy-and-market-scope.md):** geographic scope is a *present* reality, not a future expansion scenario. This does not change the recommendation — a single shared graph still wins, and it's now better supported, since a national CMO like KIPP is one organization family spanning several would-be "markets." It does mean **the sector dimension and the geographic dimension are both live simultaneously**, which strengthens the case for Market as a derived query over Organization attributes rather than a hard scope column. A KIPP NYC posting is charter + NY; the org itself spans everything.

---

## 7. Organization data quality — the identity-resolution case, evidenced

The customer list is a 20-year-old hand-maintained artifact, and it demonstrates every failure mode blueprint §4.1 anticipates. This is not a criticism of the list; it is the strongest available argument for prioritizing identity resolution in Phase 1.

**Exact duplicates differing only by URL form:**

| Records | URLs |
|---|---|
| Marion P. Thomas Charter School ×2 | `mptcs.org` / `www.mptcs.org` |
| Our World Neighborhood Charter School ×2 | `owncs.org` / `www.owncs.org` |
| Brooklyn Prospect / Brooklyn Prospect Charter School | `brooklynprospect.org` / `www.brooklynprospect.org` |
| Family Life Academy Charter School**s** / School | `flacsnyc.com` / `flacsnyc.com/` |

→ *Normalized domain is a required matching key, not a display field.*

**Same name, different domains:**

- Hebrew Public → `hebrewpublic.org` **and** `hebrewcharters.org` (two records, one organization, a rebrand captured as duplication)

→ *An Organization needs **many** domains over time, effective-dated. A single `Website` column cannot represent this.*

**Network / campus collapsed into flat records:**

| Records | Reality |
|---|---|
| Great Oaks Charter School / Great Oaks Charter School NY / Great Oaks Charter Schools | Network + campus + network, three rows |
| Lighthouse Academies / Bronx Lighthouse Charter School / Metropolitan College Preparatory Academy | Network + two campuses — and the network's domain appears as both `lighthouse-academies.org` and `lighthouseacademies.org` |
| Citizens of the World ... Williamsburg / ... New York | Campus + network |

→ *Exactly the CMO-vs-campus-vs-LEA ambiguity blueprint §4.1(a) identifies. It is already in the data.*

**Malformed and missing values:**

- `http://https://ileadschools.org` (iLEAD Schools) — double scheme
- `http://.www.rosevillecharter.org` (Roseville Community) — leading dot
- PAVE Schools — no URL at all
- `FPCHARTER.ORG`, `CAMPA CHARTER SCHOOL`, `BROOKLYN EMERGING LEADERS ACADEMY CHARTER SCHOOL` — inconsistent casing

→ *Legacy import must preserve these as **observations with provenance** (blueprint §4.4), never as asserted facts. A migration that "cleans" them silently destroys the evidence needed to decide what was actually meant.*

**Organizations that are not schools:**

BoardOnTrack, Little Bird HR (vendors) · Building Excellent Schools, Leading Educators, New Visions for Public Schools, The Urban Assembly (intermediaries/nonprofits) · District-Charter Collaborative (a NYC DOE program) · York College, CUNY (university) · Apollo After School (provider)

→ *Confirms Organization Type must span School, Network/CMO, Vendor, Nonprofit, University, Government Program. Validates modeling **Employer as a role, not a type** — several of these are customers without being schools, and York College is plausibly a future CertifiedK12 advertiser.*

### 7.1 The list is being retired — four things to handle first

*Added 2026-08-06.* Kent plans to replace the flat ~140-name list with a curated set of high-recognition logos, matching the treatment on career fair landing pages. That is the right call for conversion and is exactly what the Conversion Strategy recommends (§3, Priority 1.2). Four consequences.

**1. Capture the list before it goes. It is currently a record, not just a rendering.** Those 140 rows are the only place some of this history exists in a durable form — every duplicate, every malformed URL, every network/campus conflation documented above. Once the page is rebuilt, that evidence is gone unless it was captured first. Per blueprint §4.4, it should be imported as **`RawObservation` with source = legacy homepage and a fetch date**, never as asserted fact. This is cheap to do now and impossible to do later.

**2. Featuring must be data, not a hardcoded template list.** If the curated logos are image URLs in a Liquid template, updating the wall is a developer task forever — and it will need to differ by context (homepage vs. For Employers vs. a given fair page vs. sector-specific pages later). The right shape is an Organization query returning featured orgs, with the content layer rendering them.

This is a small but exact test of the [ADR-0001](adr/0001-orchard-core-boundary.md) boundary: **the logo wall is content; the selection is domain.** Content reads domain through a service and never stores business state. If the first thing built after ADR-0001 violates it, the boundary won't hold — so it's worth doing correctly precisely *because* it's small.

**3. Organizations need brand attributes the current model lacks.** A curated wall requires more than name and URL:

- **Logo asset** (ideally more than one format/aspect for different placements)
- **Display name** distinct from legal name — the live list contains `CAMPA CHARTER SCHOOL`, `BROOKLYN EMERGING LEADERS ACADEMY CHARTER SCHOOL`, and `FPCHARTER.ORG`, none of which are presentable at logo-wall prominence
- **Featured flag and sort weight**, per context
- **Logo usage permission** — see below

**4. The consent and current-vs-former risk gets sharper, not softer.** Buried in a list of 140, an out-of-date entry is invisible. Featured as one of fifteen logos above the fold, it is a claim. The current page is headed *"Our Customers"* and already conflates current and former — Success Academy is named in this project as a **past** customer, and it is exactly the kind of high-recognition logo a curated wall would want.

So featuring should be gated on **an active Employer/Client role** (effective-dated, per blueprint §2) rather than on ever-having-been-a-customer, with an explicit `LogoUsagePermitted` attribute alongside it. Two organizations' worth of governance today; a real problem at fifty. **[CONFIRM]** — do the current plan and fair-table terms grant logo usage rights, and do they survive lapse?

**One SEO check before deleting.** The list is ~140 outbound links carrying school names. Two possibilities and they point opposite ways: 140 follow links on the homepage dilute outbound link equity (an argument for removal), *but* the page may earn long-tail traffic for `"[school name] jobs"` queries (an argument for keeping the names somewhere). Check Search Console before dropping them. If there is traffic, the answer is both/and — curated logos on the homepage, and the full roster preserved on a dedicated page or, better, **per-organization landing pages**. The latter is worth considering independently: an org profile page is the natural public surface for the Employer Marketing Intelligence data, and it's SEO-valuable in a way a flat list never was.

---

## 8. URL inventory — input to ADR-0010 (SEO continuity)

| Path | Kind |
|---|---|
| `/` | Content |
| `/jobs` | **High-value, indexed** |
| `/events`, `/events/{slug}` (e.g. `nyc-fair`) | **High-value, indexed** |
| `/news`, `/about-us`, `/contact` | Content |
| `/JobPostingProducts/AnnualPostingPlans` | MVC — commerce |
| `/JobPostingProducts/BuyPostingCredits` | MVC — commerce |
| `/cart` | Commerce |
| `/Users/Account/LogOn?returnUrl=…` | Identity |
| `/employerapplication/` | Employer portal |
| `/terms`, `/privacy`, `/community-guidelines`, `/cookie-policy` | Legal |
| `/Themes/BoomerangTemplate/…`, `/Media/Default/…` | Assets |

**Observations:**

- **Casing is mixed** — PascalCase MVC routes (`/JobPostingProducts/…`) alongside lowercase content slugs (`/about-us`). Any change to canonical casing is an SEO event. Preserve both forms or 301 deliberately.
- **Individual job posting URLs were not observable** from the pages fetched (the `/jobs` listing is JS-rendered). These are almost certainly the highest-volume indexed URLs on the site and the single biggest cutover risk. **Retrieving the full posting URL pattern and a sitemap is a prerequisite for ADR-0010.**
- **Two sign-in entry points, one endpoint** — "Employer Sign-in" and "Candidate Sign-in" both hit `/Users/Account/LogOn`, differing only by `returnUrl`. One identity system, two presentations. Useful precedent for ADR-0003: the Person/Account split is already implicitly single-store.
- **Server-side GTM** on a first-party subdomain (`odbxiloo.charterschooljobs.com`, container `GTM-NL23RBG`). Existing analytics infrastructure worth preserving; relevant to the Intelligence context's behavioral signals.

---

## 9. The homepage audit is verified

Every substantive claim in the Conversion Strategy's §1 checks out against the live page:

- *"WANTED: Excellence and Results"* plus the three-bullet role list is candidate-facing copy above the fold. ✓
- The only employer CTA (`Post Jobs`) sits visually equal to `Find a Job` and `Career Fairs`. ✓
- The customer wall is ~140 unstyled alphabetized text links, no logos, no grouping, no CMO framing. ✓
- `Post Jobs` routes directly to pricing (`/JobPostingProducts/AnnualPostingPlans`) with no intermediate case-making page. ✓

The Conversion Strategy is a reliable source. Its two errors (§2 above, and the entitlement framing it inherited from the pricing cards) both come from **reading the pricing cards rather than the mechanism** — worth knowing when using it as input elsewhere.

---

## 10. New risk: automated employment decision tools

Not in any source document, and it needs to be on the record before AI matching is designed.

**NYC Local Law 144** regulates *automated employment decision tools* (AEDTs) used for hiring or promotion decisions in NYC — including remote roles tied to a NYC office. It has been in effect since January 2023 and enforced since July 2023. Obligations include an **independent annual bias audit** (selection/scoring rates and impact ratios across EEO-1 race/ethnicity, sex, and intersectional categories), **public posting** of the audit summary, and **at least 10 business days' advance notice to candidates** that an AEDT is in use. Penalties start at $500 per violation and escalate to $1,500 per day for continuing violations.

**Why this lands on CSJ specifically:**

1. CSJ's core market is exactly the covered jurisdiction.
2. The vision documents describe **AI-powered candidate search and matching** (Strategy p1) and **AI screening/ranking** — squarely AEDT territory.
3. The 2022 mission is explicitly selective — *"the most capable and academically accomplished"* — and operationalizing selectivity through an algorithm is the precise activity the law targets.
4. **The legal obligation falls on the employer, not the vendor.** If CSJ ships candidate ranking without supplying audit artifacts and notice mechanisms, it puts its own customers out of compliance. That is a serious commercial liability disguised as a feature.

**Architectural requirements this creates** (to be settled in ADR-0011, which should be brought forward):

- Any candidate scoring or ranking must be **inspectable and reproducible** — inputs, model version, and output recorded per decision. The §4.4 provenance chain covers this if applied to matching, not just enrichment.
- **Ranking must be separable from surfacing.** Search and filtering by employer-specified criteria is not the same as algorithmic scoring, and the boundary needs to be explicit in the design, not emergent.
- Candidate notice and the audit-artifact surface are **product features**, not compliance afterthoughts.

I am not a lawyer and this is not legal advice — whether a given feature constitutes an AEDT is a fact-specific question, and counsel should confirm scope before build. Flagging it now because the cost of designing for it is near zero and the cost of retrofitting it is not.

---

## 11. Open questions raised or changed

| # | Question | Status |
|---|---|---|
| 1 | ~~Do plan credits roll over?~~ | **Dissolved** — slots are capacity, not consumables (§1) |
| 2 | ~~Is the fair discount flat or tiered?~~ | **Resolved — flat 20%** (§3.2) |
| 3 | How do plan slots and à la carte credits interact? Which is consumed first? | **New** (§1) |
| 4 | Does a 60-day à la carte credit occupy a plan slot for 60 days, or are the systems fully separate? | **New** (§1, §3) |
| 5 | ~~What is the à la carte price schedule?~~ | **Resolved** (§3.1) |
| 5a | Is à la carte pricing above 25 units defined at all, or is 25 the cap? | **New** (§3.3) |
| 5b | ~~Should the funnel recommend à la carte to low-volume schools?~~ | **Withdrawn** — break-even is one continuously-open role; plans are the correct default (§3.3) |
| 5c | ~~Are Gold and Platinum meaningfully differentiated?~~ | **Answered — yes.** Slots are peak concurrency; both Platinum customers exceed 50 concurrent (§3.3) |
| 5d | Is a **Custom/Enterprise tier** in scope? Platinum is underpriced at the top of its range. | **New, commercial** (§3.4) |
| 5e | For a network-wide plan, is capacity **pooled, allocated per school, or hybrid**? | **New, blocking the entitlement model** (§3.4) |
| 5f | Is the 25% per-posting-month gap favouring 60-day credits intentional? | **New, minor** (§3.3) |
| 6 | ~~Is temp/contract staffing real?~~ | **Answered — yes** (§5) |
| 7 | What is the individual job posting URL pattern? Is there a sitemap? | **New, blocks ADR-0010** (§8) |
| 8 | Should internal (K12 Staffing / K12connect Search) postings be flagged and excluded from commercial reporting? | **New** (§5) |
| 9 | Is AI candidate ranking in scope for the rebuild, and has counsel reviewed AEDT exposure? | **New** (§10) |
