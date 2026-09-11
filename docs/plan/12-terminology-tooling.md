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
   allocation. SNOMED International's browser front-end runs against your own
   Snowstorm, showing the India extension. That is a materially faster working
   environment than SQL.
3. **A FHIR terminology API you will want later.** `$lookup`, `$validate-code`,
   `$expand`, `$subsumes`, `ConceptMap/$translate` — the operations the ABDM FHIR
   IG world expects. Not needed by `ddid`, but if AIIMS wants a terminology
   service for other purposes, this is the same box.

### A3. Which server

Reading "csnotc" as **CSIRO Ontoserver** — the answer below holds for any FHIR
terminology server.

| Option | Licence/cost | Ops weight | ECL | Verdict |
|---|---|---|---|---|
| **Snowstorm** (SNOMED International) | Apache-2.0, free | Heavy — JVM + Elasticsearch; realistically 16 GB RAM and ~50 GB disk for International + India extension; import measured in hours | Full ECL, authoring-grade | **Recommended.** It is what SNOMED International runs, it is free, and it handles extension dependency ordering natively |
| **Ontoserver** (CSIRO) | Commercial; free in some countries under national agreements — **unknown for India** | Lighter than Snowstorm | Full ECL, excellent FHIR conformance | Use **only if NRCeS already holds a licence**. Do not procure one for this project — [Q20](10-open-questions.md#q20) |
| **Hermes** (Wardle) | Open source, lightweight, embedded file-based store, no Elasticsearch | Very light | ECL supported | Attractive fallback if Elasticsearch operations are a real obstacle. I am moderately but not fully confident of its current extension-handling maturity — evaluate in Phase 1 before committing |
| **RF2 → PostgreSQL only** (original plan) | Free | Light | Hand-rolled | Keep as the **bulk staging layer regardless**; insufficient on its own for search and ECL |

**Recommendation: Snowstorm plus PostgreSQL staging.** Snowstorm for ECL,
description search and the browser; PostgreSQL for the relational joins,
anchors, proposals and everything in [02](02-data-model.md).

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

### A5. Is there an NRCeS-hosted option?

Worth checking in Phase 0 ([Q21](10-open-questions.md#q21)). NRCeS provides a
SNOMED CT browser for India; whether it exposes a Snowstorm or FHIR terminology
endpoint to Indian affiliates is unknown to me.

If it does: use it for **interactive curation** (zero ops cost, always current).
Still pin RF2 archives for the build — a hosted endpoint can change release
under you mid-project, which is fine for a human browsing and fatal for a
reproducible build.

### A6. Cost

| | Eng-weeks |
|---|---|
| Snowstorm standup, Intl + India extension import, dependency ordering, ES tuning | +1.5 |
| ECL expansion cache tooling, CI integration | +0.5 |
| Hand-rolled SQL closure and search work no longer needed | −1.0 |
| **Net** | **+1.0** |

Roughly break-even on effort, clearly positive on correctness and on
terminologist throughput. The hardware is the real cost: a 16 GB build machine.

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
| Engineering estimate 42 → 44 person-weeks | [07](07-effort.md) |
| Four new open questions (Q20–Q23) | [10](10-open-questions.md) |
