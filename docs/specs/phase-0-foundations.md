# Spec: Phase 0 — Foundations

**Status:** Ready to implement
**Date:** 2026-08-06
**Depends on:** [ADR-0001](../adr/0001-orchard-core-boundary.md)
**Blocks:** Phase 1 (Organizations + commercial slice)

---

## Goal

Stand up a **walking skeleton**: the smallest system that proves the [ADR-0001](../adr/0001-orchard-core-boundary.md) boundary is both *workable* and *mechanically enforced*, before any domain modelling exists to be corrupted by getting it wrong.

Phase 0 is deliberately unblocked by every open question in the project. It depends on no unresolved `[CONFIRM]` item.

## Done when

All five are true, verified by running commands — not by inspection:

1. `dotnet build` succeeds; the Orchard Core host boots and completes setup.
2. A domain module owns its own `DbContext`, its own SQL schema, and an applied EF Core migration.
3. `dotnet ef migrations add` works from the command line without error.
4. An Orchard module renders domain data **through an application service**, referencing it by opaque public code.
5. Boundary enforcement **fails** when a deliberate violation is introduced — including a *transitive* one — and passes when it is removed. *This is the acceptance test that matters most: an unproven guardrail is not a guardrail.*

## Non-goals

Do **not** build these in Phase 0, even if convenient:

- Any Organization modelling beyond the three-field stub in §4 — hierarchy, roles, external identifiers, and merges are **ADR-0004**, unwritten.
- Entitlements, products, plans, publications, or anything commercial.
- Authentication or authorization beyond what Orchard provides out of the box.
- Azure infrastructure, deployment pipelines, Cloudflare. Local SQL Server only.
- Admin UI beyond the single read-only page in §7.

---

## 0. Before you start — environment and repository

Development is on **Windows**. Nothing in this spec is platform-specific, but two setup facts are.

### 0a. The repository must be git on local disk — not in cloud storage

`docs/` and `CLAUDE.md` currently live in a Google Drive folder. **A .NET solution must not.** `bin/`, `obj/`, and Orchard's `App_Data/` churn constantly; a sync client will fight MSBuild for file locks, and Drive's on-demand placeholder files break the build in ways that look like compiler bugs.

Setup:

1. `git init` a repository on **local disk** (e.g. `C:\dev\csj`). Not in a synced folder.
2. Copy `CLAUDE.md` and the whole `docs/` tree in, and commit as the first commit. `docs/` is declared source of truth in `CLAUDE.md`, so it must be versioned alongside the code — from here, **git is canonical and the Drive copy is history**.
3. Create a **private** remote and push.
4. `.gitignore` must cover `bin/`, `obj/`, `App_Data/`, `*.user`, and `.vs/` before the first build, or Orchard's setup artefacts and a SQLite/SQL Server file may get committed.

### 0b. Prerequisites

| Tool | Note |
|---|---|
| .NET SDK | Version per §1. Pin in `global.json`. |
| SQL Server | LocalDB, Developer Edition, or Docker — any is fine. Record the choice and the connection string shape in the README. |
| `dotnet-ef` | `dotnet tool install --global dotnet-ef` |
| git | Configure `core.autocrlf` deliberately; add a `.gitattributes` normalising line endings so cross-platform diffs stay clean. |

**Acceptance:** `git log` shows the docs commit; `dotnet --version` matches `global.json`; a trivial `dotnet new console` builds and runs in the repo root and is then deleted.

---

## 1. Pin versions

**Verify before writing anything else — do not assume.**

| Item | Value | How to verify |
|---|---|---|
| Orchard Core | **3.0** (stable, `release/3.0`) | [repo README](https://github.com/OrchardCMS/OrchardCore) — the docs site lags |
| Meta-package | `OrchardCore.Application.Cms.Targets` | NuGet |
| Target framework | **Determine from the OC 3.0 package** | `dotnet add package` then inspect resolved TFM |
| .NET SDK | Match the TFM | pin in `global.json` |

Orchard Core 2.1 supported both `net8.0` and `net9.0`; 3.0's floor is unconfirmed here. **Resolve it from the package and record the answer in ADR-0001**, replacing this note.

**Acceptance:** `global.json` exists and pins the SDK; `dotnet --version` inside the repo matches it.

---

## 2. Repository layout

```
/
├── CLAUDE.md
├── global.json
├── Directory.Build.props            TFM, nullable, warnings-as-errors, LangVersion
├── Directory.Packages.props         central package management — all versions here
├── .editorconfig
├── .gitignore                       dotnet + rider/vs + App_Data
├── CSJ.sln
├── docs/
├── src/
│   ├── CSJ.Web/                     Orchard Core host — the only runnable project
│   ├── Modules/
│   │   └── CSJ.Modules.Organizations/    Orchard module: routes, views, presentation
│   └── Domain/
│       ├── Directory.Build.props    boundary guard (§6) — applies to all domain projects
│       ├── CSJ.Domain.Abstractions/ shared kernel: base entity, IClock, public-code gen
│       └── CSJ.Domain.Organizations/ entities, DbContext, migrations, application services
└── tests/
    ├── CSJ.ArchitectureTests/       boundary enforcement as a test
    └── CSJ.Domain.Organizations.Tests/
```

**Rules encoded by this layout:**

- `src/Domain/**` is the protected region. Its own `Directory.Build.props` carries the guard, so a new domain project inherits enforcement **by being created in the right folder** — no per-project opt-in to forget.
- `CSJ.Modules.*` may reference `CSJ.Domain.*`. **Never the reverse.**
- `CSJ.Web` references modules; it does not reference domain projects directly.

**Central package management** (`Directory.Packages.props`) is required, not optional: it puts every version in one file, which makes the Orchard version pin a single line and makes an accidental `OrchardCore.*` addition to a domain project visible in review.

**Acceptance:** `dotnet build` succeeds. `grep -r "Version=" src --include=*.csproj` returns nothing (all versions centralised).

---

## 3. Orchard Core host boots

`CSJ.Web` references `OrchardCore.Application.Cms.Targets`, configured for **Decoupled CMS** mode.

- SQL Server connection string in user secrets, not `appsettings.json`.
- Complete first-run setup; commit the resulting recipe if one is produced.
- Confirm the admin shell is reachable.

**Acceptance:** `dotnet run --project src/CSJ.Web` serves the site; `/admin` authenticates.

---

## 4. First domain module

`CSJ.Domain.Organizations` — plain class library. **No Orchard packages.**

### Entity — deliberately minimal

```
Organization
  Id           int, identity, PK          internal only, never exposed
  PublicCode   string(12), unique, indexed opaque cross-boundary key (ADR-0001 rule 2)
  LegalName    string(256), required
  CreatedAt    datetime2, UTC
```

> **This is plumbing, not a model.** ADR-0004 defines what an Organization *is*. Do not add hierarchy, roles, types, external identifiers, addresses, or merge support. Expect this migration to be squashed before Phase 1 ships. `LegalName` and `PublicCode` are the only fields certain enough to commit to — `PublicCode` because [ADR-0001 rule 2](../adr/0001-orchard-core-boundary.md) requires an opaque key for cross-boundary reference, and it is exactly what this phase must prove.

### DbContext

- `OrganizationsDbContext`, schema **`Org`**.
- Migrations history table **inside the schema**: `Org.__EFMigrationsHistory`. Without this, contexts collide in the shared database — the failure surfaces only when the *second* context is added, which is far too late.
- Configure with `IEntityTypeConfiguration<T>`, one file per entity.
- `PublicCode` generated in the domain (URL-safe, unambiguous alphabet — no `0/O`, `1/l`). Not a GUID: it appears in URLs.

**Acceptance:**

```bash
dotnet ef migrations add InitialCreate \
  -p src/Domain/CSJ.Domain.Organizations -s src/CSJ.Web
dotnet ef database update \
  -p src/Domain/CSJ.Domain.Organizations -s src/CSJ.Web
```

Then confirm in SQL Server that `Org.Organizations` and `Org.__EFMigrationsHistory` exist, and that Orchard's `Document` table is untouched in its own schema.

---

## 5. EF Core design-time fix

The known failure ([#9882](https://github.com/OrchardCMS/OrchardCore/issues/9882)) is *"Unable to create an object of type 'ApplicationDbContext'"* — EF's design-time tooling can't construct the context from an Orchard host.

**Primary fix — `IDesignTimeDbContextFactory<OrganizationsDbContext>`** in the domain project, reading the connection string from environment or user secrets.

Prefer this over the `CreateHostBuilder`/`IHostBuilder` workaround: it is explicit, independent of Orchard's `Program.cs` shape, and unaffected by host changes across versions. Apply the `CreateHostBuilder` change only if the factory alone proves insufficient.

**Do this before the first migration.** Hit cold, it reads as a dead end rather than a five-minute fix.

**Acceptance:** the `dotnet ef` commands in §4 run from a clean checkout with no manual intervention.

---

## 6. Boundary enforcement — the critical deliverable

**Two layers, both cross-platform, no shell scripting.**

*Revised 2026-08-06: an earlier draft specified `scripts/check-boundaries.sh` as a middle layer. Development is on Windows, and a bash script there means either Git Bash (fragile) or a duplicate PowerShell version (drift). Everything the script did is better done by assembly-closure inspection in C# — which also catches transitive references more reliably than parsing `dotnet list package` output. One fewer language, one fewer moving part, per right-sizing.*

### 6a. Build-time guard

`src/Domain/Directory.Build.props`, applying to every project in the folder:

```xml
<Target Name="GuardOrchardBoundary" BeforeTargets="Build">
  <ItemGroup>
    <_ForbiddenRef Include="@(PackageReference)"
      Condition="$([System.String]::Copy('%(Identity)').StartsWith('OrchardCore'))" />
  </ItemGroup>
  <Error Condition="'@(_ForbiddenRef)' != ''"
    Text="ADR-0001: CSJ.Domain.* must not reference OrchardCore.* — found @(_ForbiddenRef)" />
</Target>
```

Also add a companion `Error` for any `ProjectReference` pointing outside `src/Domain/` — a domain project must never reference a module or the host.

Fails in the IDE, immediately, with the ADR number in the message. Catches **direct** references only.

### 6b. `CSJ.ArchitectureTests`

Assembly-level, catching everything 6a cannot — including transitive references, which are the real risk (a domain project referencing `CSJ.Domain.Abstractions` which references Orchard):

- **Walk the full assembly closure** of every `CSJ.Domain.*` assembly; fail if any referenced assembly is named `OrchardCore*`. This is the load-bearing test.
- No public type in `CSJ.Domain.*` exposes a type from an `OrchardCore*` assembly in its signature.
- Every `CSJ.Domain.*` assembly declares exactly one `DbContext`.
- Tag these `[Trait("Category","Architecture")]` so CI can run them first.

Reflection over `Assembly.GetReferencedAssemblies()` recursively is sufficient; no third-party architecture-test package is needed, and adding one would spend right-sizing budget for nothing.

### Acceptance — prove it fails

**This is the acceptance criterion for the whole phase.** A guardrail never observed failing is not known to work.

1. Add `<PackageReference Include="OrchardCore.ContentManagement" />` to `CSJ.Domain.Organizations`.
2. Confirm **both** layers fail: `dotnet build` (6a) and `dotnet test --filter Category=Architecture` (6b).
3. Then test the transitive path specifically: remove it from `CSJ.Domain.Organizations`, add it to `CSJ.Domain.Abstractions` instead. **6a will pass and 6b must fail.** This is the case that matters — it is the one a documented convention would miss.
4. Remove it. Confirm both pass.
5. Record all three outcomes in the commit message.

---

## 7. Presentation slice — proving the boundary is workable

Enforcement proves the boundary can't be crossed. This proves work can still be done without crossing it.

`CSJ.Modules.Organizations` — an Orchard module that:

- Defines an application service interface in `CSJ.Domain.Organizations` (e.g. `IOrganizationQueries.GetByPublicCodeAsync`) returning a **DTO, not an entity**.
- Registers the domain `DbContext` and services in the module's `Startup`.
- Serves `/organizations/{publicCode}` via an MVC controller or Razor Page, rendering `LegalName`.
- Returns 404 for an unknown code.

**What this proves:** the domain project stays Orchard-free while an Orchard module consumes it; the dependency points one way; the cross-boundary reference is the opaque `PublicCode`, never an integer PK; and DI composition across the seam works — the thing most likely to be quietly awkward.

**Acceptance:** seed one Organization; `curl localhost:5000/organizations/{code}` returns its name; an unknown code returns 404; `CSJ.Domain.Organizations.csproj` contains no Orchard reference.

---

## 8. CI

Assumes GitHub Actions (swap for Azure Pipelines if the repo lives elsewhere — the steps are identical):

```yaml
- dotnet restore
- dotnet build --no-restore -warnaserror
- dotnet test --no-build --filter Category=Architecture    # boundary — fail fast
- dotnet test --no-build --filter Category!=Architecture
```

Architecture tests run **first and separately**, so a boundary violation fails with an unambiguous message instead of being buried in a general test run.

Runner: `windows-latest` matches the dev machine. `ubuntu-latest` is cheaper and faster, and nothing in Phase 0 is platform-specific — but only switch once the build is known good on both, or you'll be debugging two things at once.

**Acceptance:** CI green on a clean branch; CI red on each of the three deliberate violations from §6.

---

## 9. Close-out

When the five criteria in **Done when** pass:

1. Update `CLAUDE.md` §Verification — replace the placeholder block with the real commands.
2. Update [ADR-0001](../adr/0001-orchard-core-boundary.md) with the resolved target framework and any deviation discovered while building.
3. If anything in ADR-0001's boundary rules proved impractical, **write a superseding ADR** — do not adjust the code to fit a rule that doesn't work, and do not quietly relax the rule.
4. Record the §6 failure test outcome.

---

## Notes for the implementer

- **Where a decision seems needed and isn't in `docs/`, stop and raise it.** Phase 0 is intentionally decision-free; encountering a real design choice means the scope has drifted.
- **Resist scaffolding beyond this spec.** Generic repositories, mediator pipelines, and abstraction layers are all plausible here and all forbidden by right-sizing (`CLAUDE.md`). One developer, one deployable, one database.
- **The `Org` schema name is deliberately short.** Schemas appear in every query; `Org`, `Commerce`, `Hiring` beat `Organizations`.
- If the ADR-0001 boundary turns out to be genuinely painful to work within during this phase, that is **valuable early evidence** — say so. It is far cheaper to reconsider now than after eight modules depend on it.
