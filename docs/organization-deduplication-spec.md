# Organization Deduplication — Requirements for the Orchard Core Rebuild

Charter School Jobs / K12Connect, Inc.
Supersedes the earlier Orchard 1.10 implementation spec.

## 1. Scope decision

The Free plan (1 concurrent open position, no transaction) launches on **Orchard Core**,
not on the current Orchard 1.10 system. Duplicate prevention exists to make that lead-gen
strategy viable, so it launches with it.

**Nothing in this document is built in Orchard 1.10.** A sidecar table, projection
handler and typeahead in the legacy stack would serve a signup flow that never ships
there. The earlier spec's module layout, content handler, ambient-transaction plumbing
and merge UI are all withdrawn.

Two deliverables remain:

1. **Requirements for the rebuild** — entity model, matching, signup flow, free-plan
   fences. Sections 3–8.
2. **A migration-time deduplication** — run the matching logic once, offline, against the
   current database to decide what carries over. Section 9.

The Phase 0 findings about the current system are retained, but reframed: they now
describe **what the migration must carry**, not what a merge must re-point.

## 2. What the data actually shows

Measured on the live database, October 2026.

Distinct email domains spanning more than one published `EmployerAccount`, free-mail
excluded:

| Accounts per domain | Domains | Accounts involved |
|---|---|---|
| 2 | 72 | 144 |
| 3 | 26 | 78 |
| 4 | 10 | 40 |
| 5 | 6 | 30 |
| 6 | 1 | 6 |
| 7 | 5 | 35 |
| 8 | 1 | 8 |
| 9 | 1 | 9 |
| 10 | 1 | 10 |
| 12 | 1 | 12 |
| **Total** | **124** | **372** |

**248 accounts in excess of one per domain**, and 72 domains sitting at exactly 2 — the
signature of two people from one school each signing up. A federated network would spread
across 4, 8 or 20 campuses rather than clustering at 2.

This is a **floor**. It cannot see accounts whose users all registered with personal
addresses, one organization split across two domains, or legacy-only `Employer` rows
(§10) that have no Orchard users at all.

**Against roughly 30 accounts that have purchased anything in the past 24 months.**

That ratio is what sets the scope. The duplicate population is overwhelmingly dormant
records, so merge-in-place tooling — preview, undo, transactional three-layer re-pointing
— is not worth building. The handful of duplicates that hold real money can be reconciled
by hand. Everything else is resolved by deciding what to migrate.

## 3. Root cause

The model is not the problem. An `EmployerAccount` already holds many users, and the UI
already supports adding them.

Duplicates come from two causes:

**Cause A — the new person doesn't know an account exists.** Colleagues didn't compare
notes. The account is live and its admins are reachable.

**Cause B — the person who created the account has left.** This is the more common case.
The new hire has no credentials, no one to ask, and often no awareness an account ever
existed. The account still holds the plan, credits, posting history and event
registrations, but nobody at the school can reach it.

Cause B defeats a naive request-to-join design: routing approval to the account's admins
mails a departed employee, the request expires, and the user creates the duplicate anyway.
§7.2 handles it as a first-class flow.

In both cases the creation path compounds the damage by taking the organization name as
free text, so the duplicate is not recognizable as one afterward.

## 4. Entity model for the rebuild

**Organization** — the hiring entity. Owns plan, credits, billing contact, event
registrations, posting history. Lifecycle status: `prospect` | `claimed` | `verified` |
`customer` | `inactive`.

**User** — a person with a login email.

**Membership** — the association between a user and an organization, **carrying the role**.

**Site** — a campus under an organization, with its own name and address.

**OrganizationDomain** — one organization to many domains; a domain may map to several
organizations (federated networks), so uniqueness is on the pair, not the domain alone.

**OrganizationAlias** — former and alternate names, so a near-miss cannot recur.

### 4.1 Put the role on the membership

In the current system roles come from `Orchard.Roles` and are **global to the user** —
"admin of organization X" is the conjunction of holding `EmployerAccountAdmin` and being
associated with X. That works only because a user belongs to exactly one account.

The rebuild should put the role on the membership record. It makes "who may approve a
request for X" a lookup rather than a join, makes "every organization keeps one admin"
enforceable, and removes the bug where demoting someone in one organization demotes them
everywhere. It also leaves room for a user belonging to several organizations — a network
HR director, a consultant — without requiring it on day one.

## 5. Name normalization

Deterministic, applied identically at write and match time.

```
1. Trim, collapse internal whitespace, lowercase (invariant).
2. Unicode normalize to NFKD, strip diacritics.
3. Replace "&" with "and".
4. Remove punctuation except internal hyphens in hyphenated words.
5. Expand common abbreviations (standalone tokens only):
       cs -> charter school      hs -> high school
       ms -> middle school       es -> elementary school
       acad -> academy           prep -> preparatory
       st -> saint (leading token only)
6. Remove stopword tokens anywhere:
       the, a, an, of, for, at, inc, incorporated, llc, corp, corporation,
       nonprofit, non-profit
7. Remove sector-noise tokens ONLY if the result is non-empty:
       charter, school, schools, public, academy, academies, network, org,
       organization, foundation, institute
8. Collapse whitespace, trim.
```

| Input | Normalized |
|---|---|
| `Great Oaks Charter Schools` | `great oaks` |
| `Great Oaks Charter School - NY` | `great oaks ny` |
| `Hebrew Public` | `hebrew` |
| `Hebrew Public Charter Schools for Global Citizens` | `hebrew global citizens` |
| `Harlem Children's Zone Promise Academy` | `harlem childrens zone promise` |
| `KIPP NYC` | `kipp nyc` |
| `Uncommon Schools, Inc.` | `uncommon` |

Names reducing to a single short token need a higher bar — §6.3.

## 6. Matching

Runs in memory. Organization count is in the low hundreds, on the order of 10^4 after any
national import. Do not attempt it in SQL.

### 6.1 Signals

**Domain (`d`)** — signup email domain against stored organization domains.
- Resolves to exactly one organization: decisive, 1.0.
- Resolves to several: **not decisive.** Return all as candidates ranked by the remaining
  signals. This is the federated-network case; auto-confirming the first would attach the
  user to the wrong sibling.
- Free-mail (gmail, yahoo, outlook, hotmail, icloud, aol, proton, live, msn): 0.0 and
  flag for review. Never stored as an organization domain.

**Name (`n`)** — max of Jaro-Winkler on the full normalized strings (prefix-weighted;
organization names share leading tokens far more often than trailing ones) and Jaccard on
the normalized token sets. Max rather than blend: the first handles typos and inflection,
the second handles reordering and extra qualifiers. Compare against aliases too.

**Website host (`w`)** — registrable domain of the organization's website. Requires an
ATS blocklist, since many career-page URLs point at hosted applicant tracking systems
rather than the school's own domain:

```
myworkdayjobs.com, workday.com, greenhouse.io, boards.greenhouse.io,
lever.co, jobs.lever.co, icims.com, applytojob.com, bamboohr.com,
paylocity.com, adp.com, taleo.net, jazzhr.com, breezy.hr,
smartrecruiters.com, recruiting.paylocity.com, schoolspring.com,
applitrack.com, frontlineeducation.com, indeed.com, linkedin.com,
googleusercontent.com, sites.google.com
```

A blocklisted host yields no signal and is never stored as an organization domain —
`greenhouse.io` mapping to one organization would be actively wrong.

**Phone (`p`)** — all phone fields normalized to last-10 digits, deduplicated into a set;
match is non-empty intersection. Weight below instinct: a mobile number belongs to a
person, and the same HR director working for two organizations in sequence is a real
pattern in this sector. Discard placeholders (all same digit, `5551212`, under 10 digits).

**Location (`a`)** — normalized street address plus 5-digit postal code, or coordinates
within 100m. **Treat 0/0 coordinates as null** — otherwise every unset organization sits
at the same point in the Gulf of Guinea and matches every other one exactly.

*Co-location caveat.* In New York City, where the customer base is concentrated, distinct
charter schools routinely share a DOE building and therefore an address and coordinates.
Two organizations at one address is normal here. Location is corroborating only, capped
low, and can never push a pair to Confirmed.

**Campus overlap (`c`) — organization-to-organization only.** Two organizations whose
site sets contain the same campus (normalized name plus street address) are almost
certainly the same organization. Campus names are specific enough to beat an
organization-level address match, and it survives co-location: two different
organizations in one building have campuses at the same address under *different* names.
Not available at signup, since a new user has entered no campuses — so it applies to the
migration analysis (§9), not to the signup resolver.

### 6.2 Scoring

Decisive signals short-circuit. Everything else is a weighted average **renormalized over
the signals actually present for that pair**, since address, phone and website are
frequently blank.

```
if (d == 1.0) return 1.0

weighted = n * 0.60
total    = 0.60

if (w available) { weighted += w * 0.20; total += 0.20 }
if (p available) { weighted += p * 0.15; total += 0.15 }
if (a available) { weighted += a * 0.10; total += 0.10 }

score = weighted / total

if (both states present && differ) score *= 0.6
```

Renormalizing matters: without it, a pair with no website, phone or address caps at 0.60
and can never reach Confirmed no matter how exactly the names match. With it, a name-only
comparison scores exactly `n`, so auto-confirming on name alone demands a near-exact
match — the intended bar.

The cross-state penalty is a multiplier rather than a filter because networks legitimately
span states (KIPP, Uncommon, Great Oaks).

### 6.3 Thresholds

```
score >= 0.92   -> Confirmed
0.62 - 0.91     -> Candidates
< 0.62          -> NoMatch (still show top 3)
```

**Short-name guard:** a normalized name of a single token under 8 characters raises
Confirmed to 0.97 and the Candidates floor to 0.75. `hebrew`, `uncommon`, `kipp` are too
short to auto-confirm on name alone.

Starting points. Calibrate against the §9 labelled set before launch.

## 7. Signup resolution

The organization picker is a **typeahead over existing organizations**, not a free-text
field. Free-text entry is where nearly all duplicates come from, and this single change is
the bulk of the prevention value.

```
         [ user enters name, email, location ]
                          |
                     Resolve()
                          |
   +----------------------+----------------------+
   |                      |                      |
Confirmed            Candidates               NoMatch
   |                      |                      |
status claimed?   show list + explicit      show top 3 +
  /       \        "none of these"          "none of these"
yes        no            |                        |
 |          |      user picks --> Confirmed       |
JOIN      CLAIM          |                        |
REQUEST              user rejects -------------> CREATE
                                            (flagged for review)
```

**Only published, active organizations appear in the picker.** An organization with no
postings, no credits, no membership and no login in two years does not belong in a
typeahead whether or not it is a duplicate. This is the cheap substitute for merging the
dormant backlog — a filter, not a project.

"None of these is my organization" is a **separate, deliberate second click**, never a
link styled as an afterthought.

### 7.1 Join request (Cause A)

No create button is rendered. The user requests access; live admins approve.

Routing is decided by an **admin liveness check**, not a fixed workflow:

```
liveAdmins = admins with a login in the last 6 months
             [ and no address on the mail provider's suppression list ]

if (liveAdmins.any)  -> peer approval
else                 -> orphaned account recovery (§7.2)
```

Expire peer requests after **3 days**, not 7 — a stalled request is the exact moment a
user gives up and creates the duplicate — and on expiry escalate to §7.2 rather than
dropping it. Copy support on every request.

The wording should not promise that an administrator will respond, because under Cause B
there may be nobody there.

### 7.2 Orphaned account recovery (Cause B)

The account holds real history but nobody reachable can get into it.

**Domain match substitutes for admin approval.** If the requester's email domain matches
a domain on the account — harvested from the departed user's own login email — they are
almost certainly a legitimate colleague. That turns the hardest case into the most
verifiable one.

```
if (requester domain matches an account domain
    and domain is not free-mail
    and account has no live admin)
      -> auto-grant the standard posting role
      -> notify support (informational, not a gate)
      -> log as orphan-recovery
else  -> support review queue, same business day
```

- **Never block the user meanwhile.** They proceed to post and transact immediately.
  Blocking revenue to prevent a duplicate is the wrong trade, and it is also what pushes
  them to create one.
- Grant the posting role, never admin, on automatic recovery. A human promotes.
- Log every automatic grant. The population is small enough to audit.
- Free-mail requesters always go to the support queue.

### 7.3 Preventing Cause B

**Orphan detection report.** Organizations where no user has logged in within 12 months
(null counts as inactive). Run monthly. This doubles as a sales list — an account whose
admin left is a lapsed customer with a known school and known history, and a
re-engagement email to a fresh contact there is a warm approach.

**Second-admin nudge.** Any organization with exactly one admin gets prompted to add a
second. Single-admin organizations are the ones that become orphans.

**Departure capture.** When an admin's address lands on the mail provider's suppression
list, flag the organization and prompt remaining users to nominate a new admin.

## 8. Free plan fences

Two distinct threats, deserving different force.

**Revenue leakage** — a real charter school with three openings runs three accounts to
avoid paying. Marginal cost near zero, still a warm lead, still adds inventory. Police
lightly.

**Board quality** — staffing agencies, edtech vendors, district or private schools, or
spam using free slots. This one is dangerous: the entire differentiator is that a
candidate searching the board finds charter roles and nothing else. Police hard.

Layered, in order of leverage:

1. **Organization-as-entity with the free slot allocated to the organization**, not the
   user. The second person from a domain joins rather than creating. Makes "three
   aliases, three free slots" structurally impossible rather than a policy to enforce.
2. **No free-mail domains on the free plan.** A legitimate hiring manager has a school
   address. The rare exception — a school in its authorization year on a founder's
   personal address — goes to manual review, which is correct handling anyway. Each extra
   fake account now costs a registered domain.
3. **Reduce the prize.** Paid sorts above free in search and in the candidate digest; free
   gets no school logo and no featured placement. A school running sockpuppets for three
   unbranded bottom-sorted listings is doing real work for a weak result — and is better
   handled as a sales conversation than an enforcement action.
4. **Manual review with the posting already written.** Let them complete the posting and
   submit before verification gates it. Someone who has written a full job description
   will wait; someone asked to verify first will bounce. A spammer who must write a real
   charter job posting to enter the queue has already done more work than the payoff
   justifies. Same-business-day target, or the free plan's conversion advantage is gone.
5. **Behavioral logging, reviewed retroactively** — duplicate posting text across
   organizations, shared phone or address, multiple signups from one IP, sequential
   aliases. Query monthly, looking for patterns rather than adjudicating signups.

**Fence on quality, not quantity.** Avoid expiring free listings or capping annual
postings — throughput limits reintroduce the confusion that moving to concurrent open
positions resolved.

**Soft launch:** manually review the first 50 free accounts regardless of domain match.
That is how you learn what the abuse actually looks like before automating against a
guess.

### 8.1 Loose networks

A federated network where eight schools hire independently should get eight free slots —
those are eight genuine customer relationships and eight upgrade prospects. Ask at signup
whether they hire independently or centrally, and support child organizations under a
parent. If the only way to get eight slots is to fake eight accounts, that is what will
happen, and a legitimate signal gets taught to look like abuse.

## 9. Migration-time deduplication

The backlog is resolved by deciding what carries over, not by merging in place.

**Build the matching library first**, as a standalone assembly with no framework
dependency. Point it at the current database, offline, and produce a CSV: both names, both
ids, score, per-signal contributions, activity counts on each side, and
`dbo.Employer.DateCreated` for each.

Hand-label it. Optimize thresholds for **recall in the Candidates band** — a false
candidate costs one glance, a missed duplicate costs a bad migration. Commit the
thresholds and the labelled set; they are the regression suite for the rebuild.

Then migrate survivors only:

- **One organization per cluster**, carrying the record with real activity.
- **Loser names become aliases** on the survivor, so the same near-miss cannot recur.
- **Loser domains join the survivor's domain set.**
- **Dormant records with no activity on either side are simply not migrated.** Most of the
  372 fall here.
- **Clusters where both sides hold money** — active plan, unspent credits, event
  registrations — get reconciled by hand before migration. On these numbers that is a
  short list.

A useful property: the labelled set doubles as the calibration data for the live signup
matcher, so this work is not spent twice.

## 10. What the migration must carry

Established from the current system. This is a map for the migration, not a design for the
legacy stack.

| Layer | Owns | Keyed by |
|---|---|---|
| `CSJ.Domain` (legacy Web Forms, WilsonORMapper) | `JobPosting`, `JobPostingCredit`, `EmployerMembership`, `SchoolProfile` | `dbo.Employer.uid`, bridged from `EmployerAccountPartRecord.OldCSJSystemAccountUid` |
| `NWazet.Commerce` | Order / `InvoicePart` | `InvoicePart.CustomerAccountId` = `EmployerAccount.Id` |
| `CSJ.Events` | `EmployerRegistrationPart` | `EmployerRegistrationPart.EmployerAccountID` |
| Orchard | `EmployerAccountUser` | `CommonPart.Container` → the `EmployerAccount` item; `CommonPart.Owner` → `UserPart` |

All integer FKs, no ContentPicker-in-XML, all in one SQL Server database.

Notes that bear on the migration:

- **`EmployerAccountPart`** carries `Name`, `OldCSJSystemAccountUid`, `CareerPageUrl`,
  grade range, `CityStateLocations`, `AccountStatus`, `LogoImageUrl`,
  `CompanyDescription`. **`ContactInfoPart`** carries address, state, zip, coordinates,
  four phone fields, website and email — always `ContactType = Company` on this type,
  and optional at signup, so frequently blank.
- **Emails come from `UserPart.Email`**, reached via `CommonPart.Owner`. Not from
  `vw_aspnet_MembershipUsers`, which covers only users in ASP.NET Membership and skews to
  older accounts. The legacy view's `CreateDate` is still useful for dating records.
- **`Common_CommonPartRecord`** columns are `Id`, `OwnerId` (plain int), `CreatedUtc`,
  `PublishedUtc`, `ModifiedUtc`, `Container_id` — both links nullable with no FK
  constraints, so the application owns that integrity.
- **`OldCSJSystemAccountUid` is not a matching signal.** Every account gets its own legacy
  row at setup, so duplicates carry *different* UIDs. It is the join key for finding a
  record's postings and credits, and `dbo.Employer.DateCreated` dates it — nothing more.
- **Site data lives on `SchoolProfileMultipleSchoolEmployer` (subType 2) only.** The base
  `SchoolProfile` carries no name or address, so campus matching applies to subType 2.
- **Legacy-only `Employer` rows may exist** with no Orchard counterpart, created by
  `Employer.MakeNewAndPersist` from the old admin-only ASPX page. They are invisible to
  every Orchard-side query. Audit before migrating:

```sql
SELECT e.uid, e.Name, e.city, e.state
FROM   dbo.Employer e
WHERE  e.isProspect = 0
AND NOT EXISTS (
         SELECT 1 FROM <EmployerAccountPartRecord> p
         WHERE  TRY_CAST(p.OldCSJSystemAccountUid AS uniqueidentifier) = e.uid);
```

### 10.1 One thing to fix in the current system now

`AdminOld/EmployerAccounts/EmployerAccountDetail.aspx.cs` performs a physical delete with
an `ObjectGraph` cascade, and `Mapping.xml` sets `cascadeDelete="true"` on
`Employer → SchoolProfileList` and `EmployerMembership → AssociatedJobPostings`. One
click can destroy a customer's school profiles and their entire posting history,
irrecoverably.

That is a live liability independent of this project and worth removing or gating
regardless of the rebuild timeline.

## 11. Order of work

1. **Matching library** — normalization, signals, scoring, tests. Standalone, no framework
   dependency. Needed for the migration analysis and reused by the live signup matcher.
2. **Migration analysis (§9)** — run it against the current database, hand-label, settle
   thresholds, produce the canonical mapping.
3. **Rebuild** — entity model (§4), signup resolution (§7), free-plan fences (§8).
4. **Migrate** survivors per the mapping.
5. **Launch the Free plan** on a clean organization set.

Steps 1 and 2 can run at any time and do not touch the existing application. Step 1 is
the only code worth writing before the rebuild starts.

## 12. Open items

- Whether `dbo.Employer`'s physical table has columns absent from the Wilson mapping
  (`changesOnly="true"`, no DDL in the repo).
- Whether legacy-only `Employer` rows exist, and whether any hold postings or credits.
- The default in-organization role granted on access-request approval.
- Whether the national NCES/CCD prospect import happens before or after launch. It is not
  required by anything above.
