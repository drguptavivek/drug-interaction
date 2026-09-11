# 12 — Terminology Server and RxNorm: Decisions

Two questions raised after the first draft:

1. Should we run a terminology server (Ontoserver / Snowstorm) with the India
   extension loaded?
2. Should RxNorm be included?

Short answers: **a terminology server at build and curation time — yes; at
runtime — never.** And **RxNorm — yes, and with a larger role than the brief
gives it, because it may be structurally necessary rather than merely
interoperable.**

---

## Part A — Terminology server

### A1. Runtime: no. This is not a close call.

The core constraint is "no network dependency at query time". A terminology
server at runtime would breach it, and there is no compensating benefit: by the
time `kb.ddi` is built, every SCTID it needs is already resolved, interned and
indexed. Nothing in the query path needs subsumption testing or ECL.

There is exactly one place a server could creep in — the startup validation
("every mapped SCTID still resolves as active in the loaded release"). It must
not. That check reads a **local RF2 concept snapshot or release manifest**
(`snomed.release_manifest` in [05 §7](05-go-service.md#7-configuration-and-overlay-loading)),
never a server. A PHC with no connectivity must be able to run the check, and a
service that phones a terminology server on boot fails exactly where it is least
recoverable.

So: `ddid` links no terminology client, and has no terminology server config key.
This is worth stating as an explicit non-goal, because "we already run Snowstorm,
just call it" is a reasonable-sounding suggestion that would quietly destroy the
offline property.

### A2. Build and curation time: yes, and it is better than my first draft

The original plan (Phase 1) had us loading RF2 into PostgreSQL and hand-writing
the hierarchy queries: transitive closure over `Is a`, the `Is modification of`
graph, product decomposition joins. That works, and I would still keep the
PostgreSQL staging tables for the bulk relational work. But it means
re-implementing, in SQL, semantics that SNOMED already specifies — and
getting them subtly wrong is easy and quiet.

**ECL is the argument.** Nearly every retriever and consistency check in
[03](03-candidate-ranking.md) is one ECL expression:

| Need | ECL |
|---|---|
| The substance universe | `<< 105590001 \|Substance\|` |
| Moieties (no outgoing modification edge) | `<< 105590001 MINUS (<< 105590001 : 738774007 \|Is modification of\| = *)` |
| Salts/esters/prodrugs of a given moiety | `< 105590001 : 738774007 = 372756006 \|Levothyroxine\|` |
| Products containing a moiety (incl. its salts) | `< 763158003 \|Medicinal product\| : 127489000 \|Has active ingredient\| = << 372756006` |
| Fixed-dose combinations (cardinality) | `< 763158003 : [2..*] 127489000 = *` |
| Precise (salt-level) ingredient products | `< 763158003 : 762949000 \|Has precise active ingredient\| = *` |
| India-extension products only | `< 763158003 {{ C moduleId = <india module> }}` |
| Refset membership | `^ <refsetId>` |

Writing these as SQL against RF2 is perhaps 1–1.5 eng-weeks of work that is
mostly re-deriving well-specified semantics. Getting ECL for free, correctly
implemented and conformance-tested by the people who define it, is worth more
than the operational cost of running the server.

Three further benefits, in descending order of importance:

1. **Description search done properly.** Retriever R2 (trigram fuzzy match over
   FSN + synonyms) needs correct language-refset and dialect handling, acceptability
   ordering, and active-description filtering. Snowstorm's search already does
   this. A naive `pg_trgm` scan over the description table silently includes
   unacceptable descriptions and inactive synonyms, which shows up as
   plausible-looking wrong candidates at the top of the ranked list — the exact
   failure the tool exists to prevent.
2. **A browser for the terminologist.** The hard queue (band C, ~10% of molecules
   and a much larger share of difficulty) is where a terminologist earns their
   allocation. **This need is already met**: `nrces.in/bhts/browser/` is a
   **CSNOfinder** implementation — a hosted search-and-browse UI over the national
   content, including the Indian extensions. Nothing to build or host. Note that
   it is a *finder*, so it settles the browser question and leaves the ECL
   question entirely open (§A5).
3. **A FHIR terminology API you will want later.** `$lookup`, `$validate-code`,
   `$expand`, `$subsumes`, `ConceptMap/$translate` — the operations the ABDM FHIR
   IG world expects. Not needed by `ddid`, but if AIIMS wants a terminology
   service for other purposes, this is the same box.

### A3. Which server — settled

**CSNOServ is Snowstorm.** C-DAC's CSNOtk packages the same terminology server
SNOMED International develops, with the Indian extensions (AYUSH, CDCI)
pre-integrated and an Apache-2.0 licence. That collapses what I had written as a
three-way choice:

| Option | Verdict |
|---|---|
| **CSNOServ / BHTS** | **Use this.** Snowstorm capability — full ECL, dialect-aware description search — with the Indian content already loaded and the extension dependency ordering already done |
| Standing up our own Snowstorm | **Dropped.** Same software, and we would be re-importing content NRCeS has already assembled |
| Ontoserver | Dropped — no reason to procure a commercial server |
| RF2 → PostgreSQL | **Keep as the bulk staging layer**, as before |

Consequences:

- **ECL is available.** That was the whole argument for a terminology server
  ([§A2](#a2-build-and-curation-time-yes-and-it-is-better-than-my-first-draft)),
  and it is satisfied without a separate standup. [Q27](10-open-questions.md#q27)
  narrows from "does it do ECL" to the practical "what is the endpoint shape" —
  a FHIR `ValueSet/$expand` with an ECL filter, or a native ECL endpoint — which
  the ECL client needs to know regardless.
- **The 2.0 eng-weeks for a Snowstorm standup come out of the estimate.**
- **The hardware does not automatically come out.** Snowstorm means
  Elasticsearch, so a *locally deployed* CSNOServ still wants roughly 16 GB. The
  16 GB machine is avoidable only if curation uses **BHTS hosted** and the build
  consumes the ECL expansion cache (§A4) rather than a local instance. That is
  the recommended shape: hosted for humans, cache for the build, no server we
  operate.
- **Rule 4 is untouched.** None of this reaches runtime.

### A4. Reproducibility — the constraint that shapes the integration

[04](04-etl-pipeline.md) requires the build to be a pure function of pinned
files, verified by two independent builders producing the same
`artifact_sha256`. A live terminology server is not a pinned file, and pointing
the ETL at a long-lived server would break that guarantee the first time someone
reindexed it.

Integration pattern:

```
acquire ──► RF2 archives (pinned, checksummed)
              │
              ├─► PostgreSQL staging  (bulk relational, deterministic)
              │
              └─► Snowstorm, ephemeral container, pinned image digest,
                  imported from those same archives
                        │
                        ▼
                  ECL expansion cache:  { ecl_string, snomed_release,
                                          snowstorm_version, sha256, members[] }
                        │
                        ▼
                  committed to the repo, checksummed, part of the manifest
```

Rules that follow:

- Every ECL expression used by the build is **named, versioned and cached**. The
  build consumes the cache, not the server.
- A cache miss in CI is a **hard failure**, not a silent server call. Refreshing
  the cache is a deliberate, reviewed act that appears in a diff.
- The cache diff is reviewable: "this ECL expanded to 412 concepts last release
  and 417 now" is exactly the kind of terminology drift a human should see.
- Interactive curation may query a long-lived Snowstorm freely — it is a
  *proposal* tool, and every proposal is human-approved anyway.

This gets ECL correctness without making the build depend on a running service.

### A4a. BHTS is a network dependency — curation only

BHTS being hosted makes it excellent for curation and **disqualifying for two
other uses**, both of which will be tempting:

- Not in the build path. §A4's reproducibility argument applies to *any* live
  server, national or not: the build consumes a checksummed ECL expansion cache
  built from pinned RF2 archives, never a live endpoint whose release can change
  under it mid-project.
- Not at runtime, ever. "The national terminology server is right there, just
  have `ddid` call it for the startup check" is the single most plausible route
  by which the offline property gets destroyed. `ddid` has no terminology client
  and no config key for one, deliberately — see
  [05 §6](05-go-service.md#6-startup-validation).

### A5. What to check before committing (Phase 0, ~half a day)

Against the live BHTS endpoint and a local CSNOServ deployment:

| Check | Why it decides something |
|---|---|
| Does **CSNOServ's API** support **ECL**? Via FHIR `ValueSet/$expand` with an ECL filter, or a native endpoint? | ECL was the main argument for Snowstorm ([§A2](#a2-build-and-curation-time-yes-and-it-is-better-than-my-first-draft)). If CSNOServ does ECL, Snowstorm is unnecessary. **Check the API, not the browser** — `nrces.in/bhts/browser/` is a **CSNOfinder** implementation, i.e. a search-and-browse UI; browsing working tells you nothing about ECL |
| Which FHIR terminology operations — `$lookup`, `$validate-code`, `$subsumes`, `ConceptMap/$translate`? | Determines how much of the retriever layer is free |
| Is description search dialect- and acceptability-aware? | Retriever R2's correctness ([03 §3](03-candidate-ranking.md#3-stage-2--candidate-generation-recall-oriented-20-candidates)) |
| Which extension modules are loaded — AYUSH, CDCI, others? At which release? | Confirms we get CDCI drug codes without importing them ourselves |
| Rate limits, bulk suitability, auth | Whether a ~5,000-row drug master can be resolved against it in a batch |
| Can CSNOServ be deployed **offline** on institutional infrastructure? | Matters only for curation; the runtime service never calls it |

### A6. Cost

| | Eng-weeks |
|---|---|
| Snowstorm standup, Intl + India extension import, dependency ordering, ES tuning | +1.5 |
| ECL expansion cache tooling, CI integration | +0.5 |
| Hand-rolled SQL closure and search work no longer needed | −1.0 |
| **Net** | **+1.0** |

Roughly break-even on effort, clearly positive on correctness and on
terminologist throughput. The hardware is the real cost: a 16 GB build machine.

### A7. The CSNOServ sub-licence constrains *how* we may use it

**First, the reassurance, because the clause below reads worse than it is:**
building software that stores and uses SCTIDs is exactly what a SNOMED CT
Affiliate Licence exists to permit. India is a Member Territory, so that licence
is **free of cost** to Indian organisations and is obtained by registering at
MLDS. Nothing in this project's design is blocked by SNOMED licensing. What
follows is about *one route of access* — C-DAC's hosted/sub-licensed service —
used by someone who has **not** registered.

C-DAC issues CSNOServ under a SNOMED CT sub-licence. One clause is decisive for
this project:

> **4.2** The sub-licensee is not permitted the use of this software as part of a
> system that constitutes a SNOMED CT "Data Creation System" or "Data Analysis
> System" … the sub-licensee must not use CSNOServ to **add or copy SNOMED CT
> identifiers into any type of record system, database or document**.

That is a precise description of what our curation platform does: it copies
SCTIDs into `spoke_snomed.sctid`. So **without an Affiliate Licence, CSNOServ and
BHTS may be used only to explore and evaluate the terminology** — not to build
the mapping.

Clause 5 lifts the restriction entirely for SNOMED International Affiliates:
Affiliates *may* use CSNOServ as part of a Data Creation System or a Data Analysis
System. Affiliate registration via MLDS is free of cost to Indian organisations
and is a registration exercise, not a procurement one.

So the picture is:

| | Not an Affiliate | Affiliate (free, India) |
|---|---|---|
| Browse/evaluate SNOMED CT via CSNOServ or BHTS | Yes | Yes |
| Copy SCTIDs into a database — our curation platform, or the HMIS drug master | **No** | **Yes** |
| Build a Data Creation System (the HMIS adding SCTIDs to drug master rows) | **No** | **Yes** |
| Build a Data Analysis System (this DDI service) | **No** | **Yes** |
| Redistribute SNOMED **descriptions/FSNs** to non-Affiliates | No | No — separate obligation, handled by the `codes-only` profile ([08 §3.2](08-licensing.md#32-consequence-for-kbddi)) |

Note the scope: the last row is about republishing SNOMED *content*. Using
SCTIDs as codes inside a system is the licence's purpose, not an exception to it.
Note also that the **HMIS drug master coding exercise is itself a Data Creation
System** ([13 §5](13-hmis-neutral-integration.md#5-the-drug-master-coding-exercise)) — so
affiliate status covers both sides of the boundary, not just ours.

**Consequences:**

1. **Affiliate registration moves from "eventually, for distribution" to a
   Phase 0 action that gates the curation tooling itself.** It now gates Phase 2,
   not just Phase 7. It is administrative and free — the point is to do it early,
   not that it is hard. This is sharper than [08 §3](08-licensing.md#3-snomed-ct-and-the-india-national-licence) originally stated
   and the phase plan is updated accordingly.
2. **Clause 4.4** — "not permitted to distribute or share SNOMED CT Content or
   Derivatives" — is the same obligation the `codes-only` build profile and the
   public-repo `.gitignore` already implement. Good: two independent controls,
   one requirement.
3. **Clause 5.1** puts responsibility for fees for deployment in a **Non-Member
   Territory** on the Affiliate. Deployment within India is free of cost. If the
   artifact is ever offered to a neighbouring country, that must be checked
   first — worth stating in the deployment guide before someone offers it
   informally.

---

## Part B — RxNorm dropped; GSRS instead

**Decision: RxNorm and openFDA are out of scope. GSRS becomes the substance
authority.**

### B1. Why RxNorm was proposed, and why GSRS covers it

I had argued RxNorm up from the brief's "interoperability only" on three grounds.
GSRS — NCATS's Global Substance Registration System, the system that *issues*
UNIIs — answers all three more directly:

| What RxNorm was for | GSRS equivalent | Better or worse |
|---|---|---|
| UNII derivation | GSRS **is** the UNII registry | **Better** — source rather than a relay |
| `IN` / `PIN` split as an independent check on salt vs moiety | GSRS **ACTIVE MOIETY** relationship | **Better** — a regulatory substance determination, not a vocabulary convention |
| Structural `DDInter → DrugBank → RXCUI → SCTID` path | — | **Lost**, see B2 |

The second row matters most. Salt-versus-prodrug collapse is the highest
clinical-safety risk in the design ([C4](11-challenges-to-the-brief.md#c4)), and
GSRS's active-moiety relationship is a stronger second opinion on it than
RxNorm's IN/PIN was.

openFDA is dropped outright: it was scoped to curator evidence display only,
never a rule source, and cost several gigabytes for that.

### B2. What dropping RxNorm actually costs

Two things, one minor and one that needs watching.

**Minor — RXCUI in responses.** International interoperability only. The `RXCUI`
member stays in the `anchor_type` enum ([02 §5](02-data-model.md#5-anchors)) so it
can be populated later without a migration, but nothing produces it.

**Watch this — the SNOMED side of the UNII anchor.** The third-anchor check works
by deriving UNII **independently down each spoke** and comparing. GSRS gives the
DDInter side cleanly (drug name → GSRS substance → UNII). The SNOMED side needs
`SCTID → UNII`, and there were two routes: a SNOMED UNII map refset, or
`SCTID → RXCUI → UNII`. Dropping RxNorm removes the second.

So **[Q22](10-open-questions.md#q22) is now load-bearing on its own**:

| Q22 outcome | SNOMED-side UNII anchor | Consequence |
|---|---|---|
| UNII map refset **present** | Structural, from the refset | No change — the anchor design works as written |
| UNII map refset **absent** | Name match: SNOMED FSN ↔ GSRS substance name | **Both spokes now derive UNII partly by name**, so agreement is weaker evidence than intended |

In the second case the third anchor is degraded rather than gone, and the
mitigation is proportionate: raise the weight of ATC set-overlap (F4) and
retriever corroboration (F5) to compensate, and route more molecules to band B
so a human adjudicates. Do **not** silently keep treating UNII agreement as the
strong signal it was designed to be.

**This makes Q22 a first-week Phase 0 item**, not a curiosity — it is a `grep`
against files already on the laptop
([`snomed-releases/README.md`](../../snomed-releases/README.md#q22--is-there-a-unii-map-reference-set)).

### B3. GSRS: deploy, or just take the data?

NCATS publishes GSRS as deployable software (`ncats/gsrs3-main-deployment`) and
the UNII substance data as downloadable files.

| | Use for | Cost |
|---|---|---|
| **Data export** (UNII list + substance relationships) | The build. Pinned, checksummed, reproducible — a running server is not a pinned file ([§A4](#a4-reproducibility--the-constraint-that-shapes-the-integration)) | Small |
| **Local deployment** | Curator lookup during Phase 3: structure search, synonym exploration, moiety relationships | Another Java/DB stack to operate |

**Recommendation: data export for the build, and deploy GSRS only if curators ask
for it.** Same pattern as CSNOServ — hosted or offline lookup for humans, pinned
files for the build. The must-have is that the export includes the **substance
relationships**, not just the flat UNII code list; the active-moiety relationship
is the whole point ([16 §3](16-source-checklist.md#3-free-no-account-fetch-when-convenient)).

### B4. Consequent changes to the ranking algorithm

Retrievers R7 (RxNorm lexical) and R8 (RxNorm structural) are **removed**;
R5 absorbs their role:

| ID | Retriever | Change |
|---|---|---|
| R5 | **GSRS** — substance name and synonym match, then UNII → SNOMED via the map refset if present, else FSN name match | Strengthened; now the primary anchor path |

Scoring features ([03 §4](03-candidate-ranking.md#4-stage-3--scoring)):

| # | Feature | Change |
|---|---|---|
| F10 | ~~RxNorm structural path corroborates~~ | **Removed** |
| F11 | RxNorm IN/PIN agrees with `derivation_kind` | **Re-based on GSRS ACTIVE MOIETY**, weight raised 8 → 12 |
| P4 | ~~Supported by R7 only~~ | **Removed** |
| P6 | ~~RxNorm treats as a distinct IN~~ | **Re-based**: GSRS records a distinct substance where SNOMED asserts a modification. Weight unchanged at −20 |
| F3 | UNII agreement | **Conditional on [Q22](10-open-questions.md#q22)** — weight 25 if the refset exists, 15 if both sides are name-derived |

Six retrievers now, not eight. The brand-name prohibition becomes moot with
RxNorm gone, but keep the principle: **no brand-name evidence from any source**,
because Indian and US brand names collide for different molecules.

---

## Summary of changes to the plan

| Change | Where |
|---|---|
| Terminology server is a **build/curation** dependency, never runtime | [05 §7](05-go-service.md#7-configuration-and-overlay-loading) — unchanged and now explicit |
| CSNOServ **is** Snowstorm; separate Snowstorm standup dropped (−2.0 eng-weeks) | [01](01-phases.md), [07](07-effort.md) |
| BHTS hosted for curation + ECL expansion cache for the build ⇒ no 16 GB machine we operate | [§A3](#a3-which-server--settled) |
| **RxNorm and openFDA dropped**; GSRS becomes the substance authority | [16](16-source-checklist.md) |
| GSRS ACTIVE MOIETY replaces RxNorm IN/PIN as the salt-collapse cross-check | [04 §3.1](04-etl-pipeline.md#31-substance-graph-and-closure) |
| Retrievers 8 → 6; F10/P4 removed; F11/P6 re-based on GSRS; F3 weight conditional on Q22 | [03](03-candidate-ranking.md) |
| **Q22 is now load-bearing on its own** — no RxNorm fallback for the SNOMED-side UNII anchor | [10](10-open-questions.md#q22) |
| Q23 (DrugBank cross-refs in RxNorm) withdrawn; Q27 narrowed to endpoint shape | [10](10-open-questions.md) |
| Engineering 46.5 → 44.5 person-weeks for the core build | [07](07-effort.md) |
