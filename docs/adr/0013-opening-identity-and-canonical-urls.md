# ADR-0013: Job Opening identity and canonical URLs

- **Status:** Accepted
- **Date:** 2026-08-06
- **Deciders:** Kent Strong
- **Related:** ADR-0010 (cutover/SEO — inherits this), ADR-0004 (organization identity), [current-state.md §3.3](../current-state.md#33-the-plan-vs-à-la-carte-economics--corrected), blueprint §4.3

## Context

The job board sorts by publication date, and republishing an Opening bumps it to the top. Reposting is not an edge case — with long time-to-fill roles in a sustained teacher shortage, a single vacancy is routinely published three or four times.

This makes URL identity a **domain** decision, not a routing detail. If each republication mints a new URL:

- Link equity fragments across near-identical pages, so the hardest-to-fill roles — the ones with the most inbound links and the longest time to accumulate them — are precisely the ones whose authority is most diluted.
- Google sees near-duplicate content at scale.
- Syndication partners (Indeed, Google for Jobs, LinkedIn) hold links that rot on every repost.
- Applications and analytics scatter across URLs that represent one vacancy.

The legacy Orchard 1.10.3 system's behaviour here is **unconfirmed** and must be established (see §Consequences). This ADR settles what the new system does regardless.

## Decision

**The Job Opening owns one permanent, canonical URL. Publications never mint URLs — they change what that URL emits.**

### 1. URL scheme

```
/jobs/{publicCode}/{slug}
```

- **`publicCode`** — short, opaque, URL-safe, assigned once at Opening creation, immutable forever. The authoritative identifier. Not the internal primary key.
- **`slug`** — decorative, derived from title + organization + location for keyword value. May change freely. A request whose slug doesn't match the current one **301s** to the canonical form.

**Why `publicCode` is not the internal PK:** a sequential integer leaks posting volume to competitors and couples public URLs to storage. A short opaque code costs nothing and avoids both.

**Why the id/slug hybrid rather than a pure slug** (`/jobs/{slug}`): a pure slug needs a slug-history table and collision handling — two "Math Teacher" openings at the same network in the same year is normal, not exceptional — and breaks on retitling. The hybrid is resilient to both at a fraction of the machinery. Google treats it identically for ranking.

**Why the organization is *not* in the path** (`/employers/{org}/jobs/{code}`): Organizations merge, split, and rebrand (blueprint §4.1(c–d)) — the customer list already contains Hebrew Public under two domains from a rebrand. Putting organization identity in the path turns every merge into an SEO event. The organization appears in the decorative slug only, where a change costs a harmless 301.

### 2. Lifecycle → HTTP behaviour

The rule that matters: **`JobPosting` structured data is emitted only while a publication is active.** Leaving stale markup on expired postings risks removal from Google's job experience.

| Opening state | HTTP | `JobPosting` markup | Sitemap | Robots |
|---|---|---|---|---|
| **Published** (active publication) | 200 | Full | Yes, `lastmod` = publication date | `index, follow` |
| **Lapsed** (between publications) | 200 | **None** | Removed | `index, follow` — retains equity for the next bump |
| **Filled** | 200 | None | Removed | `index` during a grace period, then `noindex` |
| **Cancelled / withdrawn** | 200 | None | Removed | `noindex` |

No 404s and no 410s. A lapsed Opening that will be republished next month must keep its accumulated authority; returning 404 discards it and makes the next bump start from zero. Lapsed, filled, and cancelled pages all render a clear status message plus links to similar current openings, so an inbound visitor from an old link lands somewhere useful.

### 3. Structured data

Emitted from the **current publication**, not the Opening:

| Field | Source |
|---|---|
| `datePosted` | `Publication.PublishedAt` |
| `validThrough` | `Publication.ExpiresAt` |
| `title`, `description` | `Publication` snapshot (see §5) |
| `hiringOrganization` | Opening → Organization |
| `jobLocation`, `employmentType` | Opening |
| `baseSalary` | `Publication` compensation range (see §5) |
| `directApply` | Whether application is completed on CSJ |

The same URL therefore emits a changing `datePosted` across publications — which is exactly what a repost should look like to a crawler.

### 4. Canonicalization and redirects

- Self-referencing `rel="canonical"` to `/jobs/{publicCode}/{current-slug}`.
- Lowercase paths, no trailing slash, single host. Everything else 301s.
- Slug mismatch 301s to canonical (single hop — never chain).
- **Redirects live in a table, not in configuration.** Columns: normalized source path, target, status code, reason, created-at, and legacy provenance. This is required at legacy-consolidation volume, it makes redirects auditable, and it keeps history per platform principle #4. Web.config rules cannot do any of that.

### 5. Publication as the legal record

**New York Labor Law §194-B** requires employers with four or more employees to state a good-faith wage range and a written job description in any posting for work performed at least partly in New York — or performed remotely reporting to a New York office. It has been in effect since 17 September 2023, and 2026 amendments tightened the good-faith standard (placeholder ranges such as "$1 to $1,000,000" are unlawful). **The law also requires employers to maintain a history of compensation ranges and job descriptions.**

That record-keeping duty maps exactly onto the entity this ADR already requires. Each `Publication` therefore stores an **immutable snapshot** of what was advertised: title, description, compensation range, employment type, and location, as of `PublishedAt`. Editing a live posting produces a new snapshot; it never overwrites the old one.

Two consequences:

- **Product:** publishing an Opening for a New York role should require a compensation range, with a good-faith validation that rejects implausible spans. This is a compliance guardrail CSJ can offer its customers, and a differentiator worth marketing.
- **Architecture:** the obligation is the employer's, but CSJ holds the record. Publication snapshots must be retained beyond the Opening's active life and must be exportable.

Not legal advice — counsel should confirm applicability and retention period. Flagged because the design that satisfies it is the design we already wanted.

#### Why not Orchard Core content versioning?

Raised by Kent, and worth recording because it will be asked again: Orchard Core has built-in content revision history, which looks like exactly this requirement solved for free.

**It versions on the wrong event.** Orchard versions a content item when the item is *saved or published in the CMS*. §194-B asks what was *advertised, and during which window*. Those axes don't align, and the mismatch produces errors in both directions:

- A draft edited five times before going live yields five versions, **none of which were ever advertised**. The compliance record is polluted with things no candidate saw.
- A publication that runs unchanged for 30 days and is then republished unchanged yields **zero new versions**, but is two legally distinct advertising periods. The compliance record misses a period it must contain.

Four further problems follow:

1. **It wouldn't save work.** `Publication` must exist regardless — it carries `RankedAt`, the consumed entitlement (slot occupancy or credit), duration, and expiry. Those are commerce and ranking concerns, not content. Since the row is written anyway, the snapshot is **five more columns on an insert we already perform**. Orchard versioning wouldn't replace that row; it would add a second, parallel history in a different store.
2. **Two histories that can silently disagree.** Reconciling "what did this say on 3 March?" would mean joining YesSql documents to EF Core rows — which [boundary rule 4](0001-orchard-core-boundary.md) forbids. And they *will* drift: edit the content item without republishing and the version history records a change that was never advertised.
3. **The retention guarantee is wrong.** Version history is a CMS convenience — prunable, and generally removed when the content item is deleted. A legal retention record must not be destroyable by an admin clicking Delete in the CMS. Making it safe means defeating the CMS's own lifecycle.
4. **The query shape is wrong.** *"Every compensation range this employer advertised in 2026, with dates"* is one line of SQL over `Publication` rows and an awkward traversal of versioned JSON documents. Audits and compliance requests want the former.

**We are not building a versioning system.** Compliance needs "what did it say, and when" — not diffing, restore, or a version tree. Append-only rows are the simplest possible implementation of that, and simpler than the machinery being offered.

**Where Orchard versioning *is* right:** marketing pages, news, event descriptions, and especially Terms, Privacy, and Community Guidelines — where *"what did our privacy policy say on date X"* is a real question with the same shape, versioned on edit, matching the feature's semantics exactly. Same category of requirement, different content, opposite answer. That contrast is the useful part; see [ADR-0001 §Worked example](0001-orchard-core-boundary.md#worked-example-applying-the-boundary).

### 6. Material change creates a new Opening

Reusing a URL is right when the vacancy is the same and wrong when it isn't. The boundary:

- **Same Opening** — description edits, salary updates, corrections, retitling that preserves the role ("Math Teacher" → "Middle School Math Teacher").
- **New Opening** — a different role, level, or subject ("Math Teacher" → "Principal"), or a different hiring organization.

Without this rule, employers will eventually recycle a high-authority Opening for an unrelated job, which is cloaking in effect if not intent, and it corrupts time-to-fill. Enforcement is a publish-time check on title/role change with a prompt to create a new Opening; it need not be automatic, but it must be a deliberate choice rather than a silent one.

## Options considered

| Option | Verdict |
|---|---|
| **Opening owns a permanent URL; publications mutate it** | **Accepted.** Equity compounds, syndication links never rot, bump delivers a freshness signal *on* an established URL. |
| New URL per publication | Rejected. Fragments equity across exactly the roles that most need it, creates near-duplicate content, rots syndicated links. |
| Permanent URL, but 404 when lapsed | Rejected. Discards the authority the permanent URL exists to accumulate, right before the bump that needs it. |
| Pure-slug URLs with history table | Rejected. Collision and retitling machinery for no ranking benefit. |
| Organization in the URL path | Rejected. Organization merges and rebrands would become SEO events. |

## Consequences

**Positive**

- Authority accumulates on one URL across a vacancy's whole life. A bump becomes *freshness on an established page* — strictly better than a new URL, which gets freshness with zero equity.
- Sitemap `lastmod` updates on bump, which is the mechanism that turns a bump into fast re-crawl. **This is how the bump earns its price** (see [current-state.md §3.3](../current-state.md#33-the-plan-vs-à-la-carte-economics--corrected)).
- Applications, analytics, and time-to-fill attach to the Opening and survive republication.
- §194-B record-keeping satisfied as a by-product.

**Negative / accepted**

- Slug-mismatch redirect handling and lifecycle-aware rendering are extra work on the highest-traffic page type. Worth it.
- One URL with changing content means cached social previews can go stale between publications.
- Long-lived Openings accumulate many publication rows. Immaterial at CSJ's volumes.

**Blocking follow-up — ADR-0010**

The legacy system's repost behaviour is **unconfirmed and must be established before cutover planning**. If it minted a URL per repost, ADR-0010's principal work item becomes **legacy repost-cluster consolidation**: identify legacy URLs representing one vacancy (organization + title + description hash), choose the canonical target using Search Console link and impression data, and 301 the rest. That mapping is preserved as data, not discarded after the redirects are written.

## Open questions

1. Does the legacy system mint a new URL per repost? **Blocks ADR-0010 scoping.**
2. What is the grace period before a filled Opening goes `noindex`?
3. Should `publicCode` be derived from the internal id (Sqids-style) or independently random? Derived is simpler; random is more opaque. Low stakes.
4. Retention period for Publication snapshots under §194-B — counsel to advise.
5. Do syndication partners require a stable feed identifier distinct from the URL? Likely yes; should be `publicCode`.
