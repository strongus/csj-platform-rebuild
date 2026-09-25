# CSJ Platform — working instructions

Rebuild of the Charter School Jobs® platform for K12connect, Inc. Orchard Core 3.0 / ASP.NET Core / SQL Server / EF Core / Azure / Cloudflare.

**`docs/` is the source of truth.** Read before designing; don't re-derive from code. Start with `docs/blueprint.md`, then the relevant ADR. If a change contradicts a document, **say so explicitly** and propose the doc change — never resolve the conflict silently in code.

---

## Non-negotiable invariants

Violating any of these causes architectural damage that is expensive to undo. They are not style preferences.

### 1. The Orchard boundary (ADR-0001)

We run Orchard Core in **Decoupled CMS** mode.

- **Orchard/YesSql owns:** marketing content, media, routing, themes, admin shell, users/roles/permissions, tenancy, content revision history for editorial pages.
- **CSJ domain modules own (EF Core, real relational tables):** Organizations, Party/Person, Commerce, Hiring, Events, Engagements, CRM, Intelligence.

Hard rules:

- `CSJ.Domain.*` projects **must never reference `OrchardCore.*`**. This is CI-enforced; do not work around it.
- **No foreign keys cross the boundary.** Content items never hold a domain FK; domain tables never hold `DocumentId`.
- Cross-boundary references are **by opaque business key, one direction only** — content may reference a domain entity's public identifier; domain code never queries content.
- **No transaction spans YesSql and EF Core.** If a feature seems to need one, the boundary is drawn wrong — stop and raise it.
- **No business rules in content parts, drivers, or Liquid.** Presentation may read domain data through an application service; never write it.
- One schema and one `DbContext` per bounded context. **Cross-context access goes through application services, never cross-schema joins.**

### 2. Preserve history — never overwrite

- Order lines carry **immutable price-and-terms snapshots**. Never resolve a historical price by looking up the current catalog.
- Subscription tier is **effective-dated history**, not a current-value column. Same for Organization roles, hierarchy membership, and contact relationships.
- Publications store an **immutable snapshot** of title, description, and compensation range as advertised (NY Labor Law §194-B record-keeping — ADR-0013).
- Derived/AI-generated facts are stored **separately** from human-asserted facts, always with source, method, model version, confidence, and timestamp. AI output never writes silently into a canonical field.
- Merges preserve both source records and record the merge event. Never destroy the losing record.

### 3. Entitlement resolution is one service

Nothing else in the platform asks *"does this org have a plan?"* Every access check, price calculation, and feature gate calls the entitlement service. **Scattered `if (org.HasPlan)` logic is the standard way this kind of platform becomes unmaintainable.**

Four entitlement kinds — do not collapse them:

| Kind | Mechanic |
|---|---|
| **Renewable capacity (slot)** | N *concurrent* slots; a Publication occupies one for 30 days, then auto-releases. Not a balance. "Can this org publish?" is a **concurrency check**. |
| **Consumable credit** | Durable à la carte balance, depletes permanently, SKU'd by duration |
| **Non-consumable benefit** | Pricing modifier on another product line (career fair discount) |
| **Capability flag** | Feature gate tied to a purchased unit |

Plan tiers are **data rows**, never an enum — a negotiated Custom plan must be a row, not a code change.

### 4. Ubiquitous language

Use these terms exactly, in code, schema, and UI. Full glossary in `docs/blueprint.md` §2.

| Use | Not | Why |
|---|---|---|
| **Job Opening** / **Publication** | ~~Posting~~ | An Opening is the vacancy; a Publication is one 30/60-day appearance. Reposting is normal. `Posting` is retired — it hid a real modelling error. |
| **Slot** vs **Credit** | using either for both | Capacity vs. consumable. Different mechanics entirely. |
| **Payer** vs **Beneficiary** | assuming they're the same | A network buys for member schools. `Order` carries both, **both non-null**, Beneficiary defaulting to Payer. |
| **Holder** + **scope** | "the org that owns the entitlement" | An entitlement is held by one Organization and consumed within a scope (`Self` \| `SelfAndDescendants` \| `ExplicitList`). Network pools are `SelfAndDescendants`. |
| **Organization** + **Role** | "Employer" as a type | Employer/Client/Advertiser/Vendor are effective-dated *roles* an Organization holds. |
| **Posting-month** | "postings per year" | The only unit in which slots and credits compare. Used for all utilization and pricing analysis. |
| **CSJ** | ~~CharterSchoolJobs~~ | In every namespace, project name, and schema. |

**Payer or Beneficiary? Mechanical rule, no judgement:** money questions use **Payer** (revenue, invoicing, AR, collections, tax, dunning). Everything else uses **Beneficiary** (entitlement, utilization, account management, CRM, lifecycle KPIs). Exception worth knowing: **per-school** utilization comes from consumption (`Publication → JobOpening → Organization`), not from Beneficiary — a network's pooled purchase has the network as beneficiary, not any one school. ([ADR-0008](docs/adr/0008-products-plans-and-entitlement.md))

### 5. Standing business constraints

- **Candidates never pay.** The candidate side is acquisition and data. Monetization runs through employers and institutions. Changing this requires an ADR.
- **A Job Opening owns one permanent URL** (`/jobs/{publicCode}/{slug}`). Publications change what it emits; they never mint URLs. Lapsed openings stay `200` and indexed. `JobPosting` structured data is emitted **only** while a publication is active. (ADR-0013)

---

## Right-sizing

Hundreds of organizations, low thousands of publications a year, dozens of concurrent users, **one developer**. Therefore: modular monolith, one deployable, one database. **No microservices, no message bus, no CQRS, no event sourcing, no separate analytical store, no caching layer beyond in-memory + Cloudflare.**

The binding constraint is one person's ability to hold the system in their head — not throughput. Every piece of infrastructure spends that budget. If a proposal adds a moving part, justify it against this paragraph.

---

## Process

1. **Design decisions become ADRs before implementation.** `docs/adr/` — one decision per file, append-only. Reversals get a new ADR that supersedes; the old one stays.
2. **Explain tradeoffs before proposing implementation.** Options with For/Against/Verdict, not a single recommendation.
3. **Prefer specifications over immediate code** for anything non-trivial. Reference `docs/`; don't restate it.
4. **`[CONFIRM]` markers in `docs/` are unverified inferences.** Do not implement against them.
5. **Conversational decisions don't count.** If it isn't in `docs/`, it isn't decided.

## Verification

Run before considering work complete:

```bash
dotnet build -warnaserror
dotnet test --filter Category=Architecture    # boundary — run this first
dotnet test
```

The architecture tests are the most important of these — they are the only mechanical enforcement of invariant §1, and that boundary is invisible in code otherwise. A violation compiles, passes every other test, and works. **They must exist before the second domain module does.**

Enforcement is two layers: an MSBuild target in `src/Domain/Directory.Build.props` (instant, catches direct references) and assembly-closure tests in `CSJ.ArchitectureTests` (catches transitive references, which is the real risk). No shell scripts — development is on Windows.

## Known setup gotcha

`dotnet ef migrations` fails against the default Orchard Core template with *"Unable to create an object of type 'ApplicationDbContext'"*. The template's `Program.cs` doesn't expose the host-builder shape EF's design-time tooling expects. Fix: return `IHostBuilder` from `CreateHostBuilder(string[])` (not a built `IHost`), and/or add `IDesignTimeDbContextFactory<T>` per domain context. ([OrchardCore#9882](https://github.com/OrchardCMS/OrchardCore/issues/9882))
