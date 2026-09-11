# 00 — Executive Summary

## 1. What is being built

Three artifacts, built from one pipeline, with different lifecycles:

| Artifact | Lifecycle | Who runs it | Network |
|---|---|---|---|
| **Curation platform** (Go API + Svelte UI + PostgreSQL) | Runs only during curation and at release time | AIIMS informatics, behind institutional Keycloak | Institutional LAN |
| **Build pipeline** (Python ETL) | Runs on release | CI runner with licensed source access | Internet at build time only |
| **Runtime service** `ddid` (single Go binary + sidecar KB file + overlay) | Long-lived, at every site | Tertiary and primary facilities | **None required at query time** |

The knowledge base is compiled once per numbered release, signed, and shipped as
a file the binary loads read-only. Every response carries the KB version.

## 2. Architecture in one diagram

```
   SOURCES (build time, licensed)              CURATION (institutional)
   ┌──────────────────────────┐                ┌───────────────────────────┐
   │ DDInter 2.0  (CC BY-NC-SA)│               │  proposal  (append-only)  │
   │ SNOMED CT Intl + India ext│──┐            │  review    (append-only)  │
   │ CDC-India product codes   │  │            │  reviewed_by != proposed_by│
   │ UNII / GSRS  (public)     │  ├─► Python ─►│  ───────────────────────  │
   │ WHO ATC index             │  │    ETL     │  mapping_projection       │
   │ openFDA labels (public)   │  │            │  (rebuilt, never edited)  │
   │ ONCHigh list              │──┘            └─────────────┬─────────────┘
   └──────────────────────────┘                              │ numbered release
                                                             │ + diff report
                                            ┌────────────────▼────────────────┐
                                            │  kb-<version>.ddi  (signed)     │
                                            │  mmap'd, CSR pair index         │
                                            └────────────────┬────────────────┘
                                                             │
   RUNTIME (offline, every site)                             │
   ┌─────────────────────────────────────────────────────────▼───────────────┐
   │ ddid  ── POST /v1/interactions/check                                    │
   │       ── CDS Hooks order-select / order-sign (HL7 PDDI-CDS)             │
   │       ── overlay.yaml : formulary, interruptive tier, suppressions      │
   └──────────────────────────────────────────────────────────────────────────┘
```

## 3. The mapping model, restated precisely

The brief's hub-and-spoke is correct and I am keeping it. Making it precise:

- The **hub** is an internal `ingredient` row identifying a *moiety*. Its
  identifier is minted by us and is stable forever. It is never a foreign
  identifier borrowed from a source.
- **Spoke A**: `ingredient → SNOMED SCTID` in the substance hierarchy
  (`semantic_tag = 'substance'`), pinned to a named SNOMED release.
- **Spoke B**: `ingredient → DDInter drug ID`, pinned to a named DDInter release.
- **Anchors**: `UNII` (single-valued per substance) and `ATC level 5`
  (**multi-valued** — see below). An anchor is computed *independently down each
  spoke* and then compared. Agreement is evidence; disagreement blocks
  auto-preselection and is surfaced to the checker.

One correction to the brief's anchor logic: **ATC-5 is not a function of a
moiety.** Many molecules carry several ATC-5 codes for different routes or
indications (e.g. separate codes for systemic vs topical vs ophthalmic forms).
Anchor comparison must use *set overlap* (Jaccard, or "non-empty intersection"),
not equality, or it will emit false disagreements on a large minority of drugs.
See [03](03-candidate-ranking.md#5-anchor-agreement-semantics).

## 4. Attachment level: moiety, with a route gate

Interactions attach to moiety. Salt form is an attribute of the product. Both
correct. But the brief lists route only as an iron-specific exception, and it is
not — it is the general case:

| Situation | Why moiety-level alerting is wrong |
|---|---|
| Topical ketoconazole vs systemic | Negligible systemic exposure; CYP3A4 alerts are noise |
| Ophthalmic timolol vs oral | Systemic beta-blockade *is* real — must alert. Opposite conclusion to the row above |
| Inhaled vs oral corticosteroid | Different magnitude, sometimes different direction |
| Oral vs IV iron | Chelation interactions are luminal; IV iron has none |
| Oral vs IV vancomycin | Oral is non-absorbed; systemic interactions do not apply |

Therefore `route` is a first-class attribute of `product`, and every interaction
rule carries an **applicability predicate** over `(route_a, route_b)` with a
default of "systemic × systemic". This is one schema column and one predicate
evaluation, and it removes a whole class of alert-fatigue complaints. It is not
an exception list.

## 5. Outcome taxonomy — the single most important API decision

The brief is right that `no_interaction_data` and `no_interactions_found` must be
distinct. The design must push that distinction down to the **pair**, not just
the request:

```
for each unordered pair (a, b):
    if a or b is not covered by the KB  -> pair_status = not_evaluated
    else if rule exists                 -> pair_status = interaction
    else                                -> pair_status = evaluated_no_interaction
```

A request may legitimately return `interactions_found` *and* a list of pairs that
were never evaluated. Collapsing that into a single request-level verdict is how
CDS systems tell clinicians "no interactions" about a drug they know nothing
about. Full taxonomy in [06](06-api-contracts.md#3-outcome-taxonomy).

## 6. Where I disagree with the brief

Detail and reasoning in [11](11-challenges-to-the-brief.md). Summary:

| # | Constraint in brief | My position | Severity |
|---|---|---|---|
| C1 | KB "embedded" in the Go binary | ShareAlike attaches to the distributed binary. Ship a signed sidecar. | **Blocking — legal** |
| C2 | CredibleMeds TdP list shipped in the artifact | Redistribution almost certainly not permitted. Per-site licensee overlay. | **Blocking — legal** |
| ~~C3~~ | ~~DDInter "also has drug–food, drug–disease, therapeutic duplication"~~ | **WITHDRAWN — I was wrong.** All three exist (857 DFI / 8,359 DDSI / 6,033 duplication) and the brief's DDI counts were exact. See [11 C3](11-challenges-to-the-brief.md#c3) and [15](15-content-types.md). | — |
| C4 | Salt→moiety collapse "derived, not hand-curated" | `Is modification of` covers esters and prodrugs. Derive *candidates*; adjudicate. Internally inconsistent with the maker-checker principle stated elsewhere in the same brief. | **High — clinical safety** |
| C5 | Startup validation "failing loudly on mismatch" | Fail the release in CI. At runtime, degrade with a banner and per-code `stale_code` status. A CDS binary that won't boot is a worse outcome. | **High — availability** |
| C6 | "No continuous monitoring process" | Correct for SNOMED substance concepts; wrong for the Indian formulary, which changes with every CDSCO approval and FDC launch. Needs a lightweight quarterly re-census, not a monitoring service. | **Medium** |
| C7 | DrugBank for "coverage gap-filling" | DDInter is itself largely DrugBank-derived, so the gap-fill is near-circular; and the DrugBank academic licence does not permit redistributing derived DDI content. Use the CC0 *Open Data* vocabulary subset for names/UNII only. | **Medium — legal + low value** |
| C8 | Keycloak auth | Contradicts "no network dependency at query time" at a PHC. Use offline JWT validation against a cached JWKS, plus a non-OIDC fallback credential for disconnected sites. | **Medium** |
| C9 | CDS Hooks as the HMIS integration path | Most Indian public-sector HMIS deployments are not FHIR/CDS-Hooks capable. Build REST first; treat CDS Hooks as the second surface, not the primary. | **Medium — sequencing** |
| C10 | ATC-5 as an equality anchor | Multi-valued; use set overlap. | Low — correctness |
| C11 | Route treated as an iron-specific exception | Route is the general gate. | Medium |
| C12 | "Top 50 salts per department" as the scope primitive | Frequency-ranked *salts* under-represents the drugs that matter for DDI (narrow therapeutic index, CYP/P-gp perpetrators, QT). Add a second, small, risk-ranked list. | Medium — clinical value |

## 7. Success criteria for the whole programme

These are the numbers the pilot must hit, agreed before build:

| Metric | Target | Measured how |
|---|---|---|
| Coverage: AIIMS phase-1 molecules present in DDInter | ≥ 85% | Phase 0 census |
| Mapping precision (approved mappings correct on blind re-audit) | ≥ 99% | 10% sample re-adjudicated by a third clinician |
| Recall against departmental validation set | ≥ 90% of known pairs found | Departmental validation set |
| Interruptive alerts per 100 medication orders | ≤ 2 | Shadow-mode logs, 4 weeks |
| Override rate on interruptive alerts | ≤ 50% | Pilot logs |
| p99 latency, 20-drug list, in-process | ≤ 5 ms | Load test |
| Cold start to serving | ≤ 3 s including startup validation | Benchmark |
| RSS at steady state, full KB | ≤ 512 MB | Benchmark |

An override rate above ~80% means the interruptive tier is wrong and must be
re-cut before go-live. That is a stop condition, not a metric to report.
