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

### A3. Which server — revised after the NRCeS / C-DAC findings

My first draft read "csnotc" as CSIRO **Ontoserver** and compared it against
Snowstorm. That was the wrong comparison. The actual options are India's own
national terminology services, and they change the recommendation.

| Option | What it is | Carries Indian extensions? | Ops weight |
|---|---|---|---|
| **CSNOServ (locally deployed)** | C-DAC's SNOMED CT terminology service, part of the **CSNOtk** toolkit (**Apache-2.0**, © 2014 C-DAC), progressively enriched with Indian extensions — AYUSH and the **CDCI clinical drug codes** | **Yes, pre-integrated** | Unknown; to evaluate |
| **BHTS** (`nrces.in/bhts`, API at `/bhts/api/v1/csnoserv/`) | The national **Bharat Health Terminology Service** — a hosted, FHIR-compliant terminology server with open API access | Yes | **None** (hosted) |
| **Snowstorm** (SNOMED International) | Apache-2.0, what SNOMED International runs | No — we would import them ourselves | Heavy: JVM + Elasticsearch, ~16 GB RAM |
| **Ontoserver** (CSIRO) | Commercial | No | Moderate |
| **RF2 → PostgreSQL** | Bulk staging | n/a | Light |

**Revised recommendation, in priority order:**

1. **Evaluate CSNOServ locally deployed first.** It is the natural fit: it already
   carries the Indian extensions including the CDCI drug codes, which is exactly
   the content this project needs, and it removes the extension
   dependency-ordering work budgeted for a Snowstorm import. If it supports ECL
   (§A5), it replaces Snowstorm outright.
2. **Use BHTS for interactive curation lookup.** Hosted, national, zero ops cost,
   always current. Ideal for the maker/checker/terminologist working a queue.
3. **Snowstorm as the fallback** for the build-side materialisation if CSNOServ's
   ECL support turns out to be insufficient.
4. **PostgreSQL staging in all cases**, for the relational joins, anchors and
   proposals in [02](02-data-model.md).

Ontoserver drops out of consideration — there is no reason to procure a
commercial server when the national service exists, is free to Indian users, and
is open source.

**CSNOtk is Apache-2.0** (© 2014 C-DAC), with components `CSNOLib`, `CSNOFinder`,
`CSNOServ` and `CSNOCtrl`. That is the same licence as our own `ddid`, so
deploying, modifying and integrating it raises no procurement or
copyleft question.

Note the shape of it, because it is the same split this plan makes for its own
artifact ([C1](11-challenges-to-the-brief.md#c1)): **the software is Apache-2.0,
the SNOMED CT content it serves is separately governed** by the sub-licence in
§A7. C-DAC separated code from licensed data for exactly the reason we separate
`ddid` from `kb.ddi`. Useful precedent when that decision is questioned.

**One thing the Apache licence does not make safe:** linking `CSNOLib` into
`ddid`. The constraint on the runtime is not the code licence — it is the offline
property and the SNOMED content, neither of which an Apache-2.0 header changes.
"It's open source, just embed the library" is the most plausible-sounding way to
breach §A1, and the answer is still no.

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

## Part B — RxNorm

The brief says RxNorm is "secondary, for international interoperability only."
**I would upgrade that** — and simultaneously tighten it. Both changes matter.

### B1. Why RxNorm may be load-bearing, not optional

The anchor design in [02 §5](02-data-model.md#5-anchors) depends on computing
UNII **independently down each spoke**. The SNOMED side of that was assumed to
come from a UNII map reference set in the release. I am not confident that map
exists in the International Release — it needs Phase 0 verification
([Q22](10-open-questions.md#q22)).

If it does not exist, the SNOMED-side UNII anchor has no derivation path, and the
third-anchor check — the thing that detects spoke disagreement, the control the
whole hub-and-spoke model leans on — quietly stops working.

**RxNorm supplies that path.** RxNorm carries `SNOMEDCT_US` atoms (substance
concepts are International core, so the SCTIDs are the same ones we map to) and
UNII codes as concept attributes. So:

```
SCTID ──(RxNorm SNOMEDCT_US atom)──► RXCUI ──(RxNorm UNII attribute)──► UNII
```

That is a genuine, independent derivation of the SNOMED-side UNII. It is not
"interoperability" — it is the anchor design's fallback leg, and possibly its
only leg.

### B2. RxNorm as an independent check on the riskiest decision

[C4](11-challenges-to-the-brief.md#c4) — automated salt/moiety collapse — is the
highest clinical-safety risk in the design. RxNorm has a directly relevant
structure, built by humans for the same purpose:

| RxNorm term type | Meaning | Our equivalent |
|---|---|---|
| `IN` | Ingredient | moiety (the `ingredient` hub) |
| `PIN` | Precise ingredient | salt / specific form |
| `MIN` | Multiple ingredients | FDC ingredient set |

with `has_precise_ingredient` / `form_of` relating PIN to IN.

So RxNorm's IN/PIN split is a **second, independently curated opinion** on
exactly the question SNOMED's `Is modification of` answers ambiguously. Where the
two agree, confidence rises sharply. Where they disagree — RxNorm treats
something as a distinct ingredient that SNOMED marks as a modification — that is
precisely the ester/prodrug/complex case that must not be auto-collapsed.

This is a better `derivation_kind` signal than the FSN-morphology heuristics in
[04 §3.1](04-etl-pipeline.md#31-substance-graph-and-closure), and it is cheap.
It should be added there as a feature.

### B3. RxNorm as a bridge to DDInter

DDInter is substantially DrugBank-derived and — to verify in Phase 0
([Q23](10-open-questions.md#q23)) — likely publishes DrugBank cross-references.
Recent RxNorm releases carry `DRUGBANK` as a source vocabulary. If both hold:

```
DDInter ID ──► DrugBank ID ──► RXCUI ──► SNOMEDCT_US ──► SCTID
```

A **structural** path from spoke B to spoke A that involves no string matching at
all. That is far stronger evidence than any name-similarity score, and it is a
genuinely independent retriever.

If either link fails to materialise, the path degrades gracefully to name
matching and nothing else in the design changes.

### B4. The tightening: ingredient level only, never brand

RxNorm's limitation is real and the brief is right to be cautious — it just
locates the caution in the wrong place.

| RxNorm content | Use? | Why |
|---|---|---|
| `IN` / `PIN` / `MIN` (ingredient level) | **Yes** — structural anchor, collapse check | Ingredients are chemistry, not market |
| `SCD` / `SBD` (clinical/branded drug) | **No** | US market products; irrelevant to an Indian formulary |
| Brand names (`BN`) | **Prohibited** | Indian and US brand names collide frequently for entirely different molecules. A brand-name match is a *false-evidence generator*, worse than no evidence, because it looks like corroboration |

The risk the brief senses is real, but it lives in the brand layer, not the
ingredient layer. Blocking brands and promoting ingredients gets both halves
right.

### B5. Consequent changes to the ranking algorithm

Two changes to [03](03-candidate-ranking.md):

**A new retriever, structural rather than lexical:**

| ID | Retriever | Notes |
|---|---|---|
| R8 | RxNorm structural: `DDInter → DrugBank → RXCUI → SNOMEDCT_US → SCTID`, and `SCTID → RXCUI → UNII` | High precision; no string matching; may be the primary UNII derivation |

**New scoring features:**

| # | Feature | Range | `w_i` | Notes |
|---|---|---|---|---|
| F10 | RxNorm structural path corroborates the candidate (R8) | 0/1 | **20** | Comparable to UNII agreement — it is often the same evidence |
| F11 | RxNorm IN/PIN agrees with the proposed `derivation_kind` | 0/1 | 8 | Independent second opinion on collapse |
| P6 | RxNorm treats as a distinct `IN` what SNOMED marks as a modification | 0/1 | **−20** | Strong prodrug/ester signal; blocks band A |

Existing R7 (RxNorm *name* matching) keeps its −20 penalty and its "never a sole
basis" rule. The distinction is the point: the **structural** path is strong
evidence, the **lexical** path is weak and dangerous. The brief collapsed both
into "secondary"; they deserve opposite treatment.

### B6. Licensing

| | Position |
|---|---|
| Access | RxNorm full monthly release, free, via a UMLS Terminology Services account |
| Licence | UMLS Metathesaurus Licence. RxNorm's own content (`SAB=RXNORM`) is openly redistributable with attribution; **several source vocabularies inside the release are proprietary** (e.g. MMSL, GS, NDDF, MDDB) and are not |
| What we may ship | RXCUIs and RxNorm term types. Nothing else |
| What we must not ship | Atoms from restricted sources |
| Mechanism | Same as SNOMED: filter to `SAB=RXNORM` at ingest, and add a `verify` assertion that no restricted-source atom reached the artifact |
| Attribution | NLM attribution line added to `NOTICE` |

This fits the existing compliance pattern in [08 §7](08-licensing.md#7-compliance-mechanics-not-compliance-promises)
exactly — one more row in `source_release` with `redistributable` set per source
vocabulary, and the build gate does the rest.

### B7. Cost

| | Eng-weeks |
|---|---|
| RxNorm ingest (RRF parsing, `SAB` filtering, IN/PIN graph) | +0.5 |
| R8 structural retriever + F10/F11/P6 features | +0.5 |
| **Net** | **+1.0** |

Against that: it de-risks [R3](09-risks.md) (prodrug mis-collapse, score 15) and
may rescue the entire anchor design if the SNOMED UNII refset turns out not to
exist. Clearly worth it.

---

## Summary of changes to the plan

| Change | Where |
|---|---|
| Terminology server is a **build/curation** dependency, never runtime | [05 §7](05-go-service.md#7-configuration-and-overlay-loading) unchanged and now explicit |
| Snowstorm added to Phase 1; ECL expansion cache added to the ETL | [01](01-phases.md), [04](04-etl-pipeline.md) |
| RxNorm promoted from "interoperability" to structural anchor and collapse check | [03](03-candidate-ranking.md) |
| RxNorm brand layer explicitly prohibited | [03](03-candidate-ranking.md) |
| UMLS/RxNorm licence row, `SAB=RXNORM` filter, verify assertion | [08](08-licensing.md) |
| Engineering estimate 42.5 → 44.5 person-weeks | [07](07-effort.md) |
| Four new open questions (Q20–Q23) | [10](10-open-questions.md) |
