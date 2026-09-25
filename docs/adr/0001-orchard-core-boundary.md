# ADR-0001: Orchard Core is the host, not the domain model

- **Status:** Accepted
- **Date:** 2026-08-06
- **Revised:** 2026-08-06 — verified against [docs.orchardcore.net](https://docs.orchardcore.net/en/latest/); terminology corrected to match the project's own
- **Deciders:** Kent Strong
- **Supersedes:** —
- **Target version:** Orchard Core **3.0** — stable branch `release/3.0`, meta-package `OrchardCore.Application.Cms.Targets`. Pin it; record upgrades here. *(The docs site said 2.1.5 on this date; the repository is authoritative for version.)*

## Context

The CSJ platform is being rebuilt on Orchard Core / ASP.NET Core / SQL Server / EF Core / Azure. Orchard Core is a content management system. Its native persistence layer is **YesSql**, a document store built on top of a relational database: content items are serialized to JSON and stored in a `Document` table, with separate *index tables* maintained by index providers to make querying possible.

This is a good design for content. It is a poor design for a business domain that needs:

- Enforced foreign keys and referential integrity
- Set-based queries and joins across many entities
- Temporal and provenance columns on individual fields
- Deterministic migrations of business data
- Analytical querying (marketing intelligence, reporting)

CSJ is not primarily a content site. It is a CRM + marketplace + commerce system with a marketing site attached. Modeling Organizations, Contacts, Subscriptions, Postings, and Applications as Orchard content types would directly contradict the platform principle of *normalized relational models with explicit relationships*.

Conversely, discarding Orchard Core entirely would mean rebuilding a large amount of genuinely useful infrastructure: admin shell, permissions, roles, users, tenancy, media management, routing, templating, workflows, and a CMS that non-developers can edit.

## The distinction the project itself draws

*Added 2026-08-06 after reading the official documentation. This is the most useful correction to this ADR, because Orchard already has vocabulary for the line we were drawing by hand.*

The docs open by separating two products shipped independently:

> *"Orchard Core consists of two different targets: **Orchard Core Framework** — an application framework for building modular, multi-tenant applications on ASP.NET Core. **Orchard Core CMS** — a Web Content Management System built on top of the Orchard Core Framework. […] It's very important to understand the Orchard Core Framework is distributed independently from the CMS."*

The Framework provides modularity, multi-tenancy, DI, and module loading. The **CMS** provides content items, content types/parts, and the YesSql document storage that goes with them. My original framing — "Orchard Core is a CMS whose document store fights our domain" — conflated the two and was cruder than it needed to be.

**Restated in the project's own terms:**

| Layer | Our use |
|---|---|
| **Orchard Core Framework** | Adopted wholesale — modules, DI, tenancy, configuration, module loading |
| **Orchard Core CMS** | Adopted **selectively**, for editorial content only |
| **CSJ domain modules (EF Core)** | Everything the business is about |

The boundary in this ADR is essentially the Framework/CMS seam, plus a rule about which side business entities live on. That is a far more natural line than the one I described, and it means the arrangement runs *with* the project's design rather than against it.

**The mode has a name: "Decoupled CMS."** The docs list it as one of three supported site-building strategies:

> *"**Decoupled CMS.** The site starts off blank, apart from the content management back-end. You create all the templates you need with Razor Pages or MVC actions and access your content via the content services."*

This is precisely what this ADR prescribes. Use the term — it makes the intent legible to anyone who knows Orchard, and it means we are on a documented, supported path rather than an improvised one.

## Decision

**Orchard Core hosts the application. It does not model the business.**

We keep Orchard Core, and we draw a hard boundary through the middle of the system.

### Orchard Core / YesSql owns

- Public marketing site content: pages, articles, landing pages, navigation, SEO metadata
- Media library and asset management
- Routing, theming, Liquid templates, widgets
- The admin shell and its navigation/menu system
- Users, roles, permissions, authentication, external login providers
- Tenancy infrastructure (see ADR-0002)
- Workflows, where used for content or notification orchestration
- **Content revision history** — including for Terms, Privacy, and Community Guidelines, where "what did this say on date X" is a genuine compliance question that versions on edit. See the worked example below for where this stops.

### CSJ domain modules / EF Core own

- Organizations and organization hierarchy
- Party/Person, contacts, and organization relationships
- Products, plans, entitlements, subscriptions, orders
- Job postings and applications
- Career fairs and registrations
- External data ingestion, enrichment, and provenance
- All analytics and reporting tables

These live in ordinary EF Core `DbContext`s with real relational tables, real foreign keys, and EF Core migrations, inside Orchard Core modules in the same deployable.

## Boundary rules

These rules are what make the arrangement survivable. They are not negotiable without a superseding ADR.

1. **No foreign keys cross the boundary in the database.** A content item never holds an FK to a domain table; a domain table never holds an FK to `DocumentId`.
2. **Cross-boundary references are by stable business key, in one direction only.** If a content item must reference an Organization, it stores the Organization's public identifier (a GUID or slug) as an opaque value. Domain code never queries content.
3. **No business rules in content parts, drivers, or Liquid.** Presentation may read domain data through an application service; it may never write it.
4. **No distributed transactions.** A single request must not need to write atomically to both YesSql and an EF Core context. If a workflow appears to require it, that is a signal the boundary is drawn in the wrong place — redesign, don't span.
5. **Each domain module owns exactly one schema and one `DbContext`.** SQL Server schema per bounded context (e.g. `Org`, `Commerce`, `Hiring`). Cross-context reads go through application services, not cross-schema joins.
6. **Domain migrations are EF Core migrations, checked in, and run explicitly.** They do not run inside Orchard's `Migrations` classes, which are for content type definitions only.
   - **Known friction, with a known fix.** The `dotnet ef migrations` design-time tooling fails against a default Orchard Core project template with *"Unable to create an object of type 'ApplicationDbContext'"*, because the template's `Program.cs` doesn't expose the conventional host-builder shape EF Core's tooling looks for ([OrchardCMS/OrchardCore#9882](https://github.com/OrchardCMS/OrchardCore/issues/9882)). Fix: expose `CreateHostBuilder(string[] args)` returning `IHostBuilder` (not a built `IHost`), and/or implement `IDesignTimeDbContextFactory<T>` per domain context. **Do this in project setup, before the first migration** — it is a five-minute fix that reads as a dead end if hit cold.
7. **Admin UI for domain entities is purpose-built,** not generated from content type definitions. Accept this cost; it is the price of the integrity we want.

## Worked example: applying the boundary

The seven rules say *where* the line is. This example says *how to decide* when a case looks ambiguous — and the first real one arrived within a day of writing them.

**The question (Kent, 2026-08-06):** NY Labor Law §194-B requires employers to retain a history of advertised compensation ranges and job descriptions. Orchard Core has built-in content revision history. Doesn't that solve it for free?

**The answer is no, and the reason generalizes into a test:**

> **Does the framework feature trigger on the *content* event, or on the *domain* event?**

Orchard versions on **save/publish of a content item** — an editorial event. §194-B cares about **what was advertised and during which window** — a commercial event tied to entitlement consumption. Because the triggers differ, the histories diverge in both directions: drafts edited but never published create records of things nobody saw, while a republication with no text change creates no record of a legally distinct advertising period.

When the triggers align, use the framework feature. When they don't, no amount of configuration will reconcile them, and adopting it anyway means maintaining two histories that disagree.

**The same requirement, opposite answers:**

| Requirement | Trigger | Home | Why |
|---|---|---|---|
| *"What did our Privacy Policy say on 3 March?"* | Editorial edit | **Orchard versioning** | The document changing *is* the event |
| *"What compensation range did this employer advertise on 3 March?"* | Publication window | **`Publication` snapshot rows** | The advertising period is the event; the text may never have changed |

Both are "keep a history for compliance." They land on opposite sides of the boundary, and the trigger is what decides it.

**Two supporting checks worth running on any similar question:**

- **Would the framework feature actually save work?** Here it wouldn't — `Publication` rows must exist regardless (they carry `RankedAt`, consumed entitlement, duration, expiry), so the compliance snapshot is a few extra columns on an insert already happening. A "free" feature that adds a parallel store alongside a row you're writing anyway is not free.
- **Are the durability guarantees right?** CMS version history is prunable and typically disappears when a content item is deleted. A legal retention record must not be destroyable from the CMS admin UI.

Full reasoning: [ADR-0013 §5](0013-opening-identity-and-canonical-urls.md#why-not-orchard-core-content-versioning).

## Options considered

### A. Fully Orchard-native (business entities as content types)

- **For:** Fastest path to a working admin UI; deep framework alignment; flexible schema-less evolution; free versioning, drafts, and audit of content items.
- **Against:** No referential integrity. Queries across entities require hand-built index providers and get slow and awkward fast. Reporting over JSON documents is painful. Complex relationships (organization hierarchy, entitlements, identity resolution) become unmanageable. Directly violates the normalized-relational principle.
- **Verdict:** Rejected. This is the trap that makes Orchard Core rebuilds fail at exactly the scale CSJ is aiming for.

### B. Orchard Core hosts, EF Core owns the domain (chosen)

*Post-verification: this is the **Decoupled CMS** strategy, which the project documents and supports explicitly — the option is more mainstream than the original write-up implied.*

- **For:** Full relational integrity where it matters. Orchard's infrastructure retained where it earns its keep. One deployable, one database, one deployment pipeline. EF Core is a first-class, well-understood tool for the domain work. Documented, named, supported path.
- **Against:** Two persistence stacks in one process. Two migration systems. Transaction scope must be reasoned about at the boundary. Domain admin UI is hand-built. Some Orchard features (content versioning, workflow content triggers, search indexing) do not apply to domain entities without extra work.
- **Verdict:** Accepted.

### C. Drop Orchard Core; plain ASP.NET Core + EF Core

- **For:** One stack, one mental model, minimum ceremony. Ideal for a solo developer working with AI-generated implementations.
- **Against:** Rebuild admin shell, identity, permissions, media, tenancy, and CMS. Marketing content becomes a developer task rather than a Kent task.
- **Verdict:** Rejected for now, but this remains the fallback if the boundary proves too expensive in practice. Revisit if the marketing/content surface turns out smaller than expected.

### D. Orchard Core **Framework** only, no CMS

*Added 2026-08-06 — the documentation makes this a real option that the original write-up missed.*

The Framework ships independently of the CMS, with its own sample applications, and gives modularity, DI, and multi-tenancy without content items or the CMS content pipeline.

- **For:** Orchard's modularity and tenancy without any of the persistence tension this ADR exists to manage. The boundary would be structural rather than disciplinary — there would be no content-item model to misuse.
- **Against:** Loses the admin content editing that is the main reason to adopt Orchard at all. Marketing pages revert to a developer task, which is most of the argument for option C anyway — but with the Framework's learning curve retained.
- **Verdict:** Rejected, but for a sharper reason than before: the CMS is the *point* of adopting Orchard here. If content editing stops mattering, the honest move is option C, not this. Recorded so the option isn't rediscovered as though it were new.

## Consequences

**Positive**

- The business domain is a clean relational model that AI-generated implementations (Codex) can reason about reliably from a schema.
- Marketing intelligence and analytics have real tables to query.
- Provenance, temporality, and identity resolution are expressible.
- Marketing pages remain editable by non-developers.

**Negative / accepted costs**

- Every domain module needs hand-built admin screens. Budget for this in the roadmap.
- Two migration systems must be run in the right order on deploy. Document the deploy sequence.
- Contributors must internalize the boundary. It is invisible in the code unless enforced.
- The `Document` table and domain tables sit in the same database. Fine at CSJ's scale, and it keeps backup/restore simple.

**Enforcement**

- Solution structure: `CSJ.Web` (Orchard host) / `CSJ.Modules.*` (Orchard modules, presentation) / `CSJ.Domain.*` (domain + EF Core, no Orchard references).
- `CSJ.Domain.*` projects must not reference any `OrchardCore.*` package. This is checkable in CI and is the single most valuable guardrail here — add it before writing the second module.
- **This matters more, not less, with agentic implementation.** The boundary is invisible in code: a violation of rule 1 or 5 compiles, passes tests, and works. Nothing surfaces the mistake until the coupling is load-bearing. A documented convention will eventually lose to plausible code that runs. Enforce mechanically in two layers — an MSBuild target in `src/Domain/Directory.Build.props` for direct references, and assembly-closure tests in `CSJ.ArchitectureTests` for transitive ones — and mirror the rules in `/CLAUDE.md` so they are in context every session. Specified in [phase-0-foundations.md §6](../specs/phase-0-foundations.md#6-boundary-enforcement--the-critical-deliverable).

## Open questions

- Does authentication/identity live entirely in Orchard's user store, or does the domain maintain its own `Party`/`Person` with a link to the Orchard user id? Leaning strongly toward the latter. To be settled in the Person/Party ADR.
- How is content-side search (Lucene/Elasticsearch) reconciled with domain-side search over relational tables? Likely: separate mechanisms, no unified index initially.
