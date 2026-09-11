# 14 — Phase 0 Runbook

Phase 0 exists to kill the project cheaply if the sources do not support it, and
to close the questions that change the shape of everything downstream. This is
the executable form of [01 Phase 0](01-phases.md#phase-0--feasibility-licence-clearance-coverage-census):
ordered steps, each with an owner, a duration, what it unblocks, and how you know
it is done.

**Target: 3 weeks.** Most of it is waiting on people, not on work.

## 0. Settled — do not re-open

| | Status |
|---|---|
| SNOMED Affiliate Licence + MLDS access | **Held** ([Q28](10-open-questions.md#q28)) |
| SNOMED release files | **Downloaded, on the developer laptop**, gitignored |
| India extension supplies products; International supplies substances | **Confirmed** ([Q2](10-open-questions.md#q2)) |
| CDCI is RF2 with real drug-model structure, via MLDS | **Confirmed** ([Q4](10-open-questions.md#q4)) |
| Terminology server: prefer CSNOServ / BHTS; Snowstorm fallback; **never at runtime** | **Decided** ([12 §A3](12-terminology-tooling.md#a3-which-server--revised-after-the-nrces--c-dac-findings)) |
| KB ships as a signed sidecar, not `go:embed` | **Decided** ([11 C1](11-challenges-to-the-brief.md#c1)) |
| CredibleMeds not shipped; site-supplied overlay | **Decided** ([11 C2](11-challenges-to-the-brief.md#c2)) |
| HMIS-neutral; SCTIDs supplied by the caller | **Decided** ([13](13-hmis-neutral-integration.md)) |

The eleven standing rules are in [`CLAUDE.md`](../../CLAUDE.md). If one looks
wrong, argue it against the document that sets it out — don't reverse it quietly.

---

## 1. Critical path

Everything else can run in parallel. This is the chain that decides whether the
project proceeds:

```
P1 CDCI access ─► P2 unpack + checksums ─► P3 inspect RF2 ─┐
                                                            ├─► P7 COVERAGE CENSUS ─► GO/NO-GO
P5 DDInter acquired + contents verified ────────────────────┘
```

**P7 is the gate.** [R1](09-risks.md) — DDInter coverage of the Indian formulary
— is the highest-scored risk in the plan, and it is the one thing that cannot be
designed around.

---

## 2. Steps

### P1 — Confirm CDCI access on MLDS
**Owner** Project lead · **1 day, then wait** · **Blocks** P2, and the whole
product layer

The only remaining licensing item. An International Edition download does not
imply access to the national packages; CDCI is listed separately at
`nrces.in/services/national-releases`.

- [ ] CDCI (Common Drug Codes for India, Terminology Integrated Package) visible
      and downloadable under the existing MLDS registration
- [ ] If not, requested — allow 4–5 business days
- [ ] Note the CDCI version and the International Edition version it pairs with

**Done when** the CDCI package is on disk alongside the International Release.

### P2 — Unpack, checksum, pin
**Owner** Engineering · **0.5 day** · **Blocks** P3, all ETL

- [ ] Unpack into `snomed-releases/` per [that README](../../snomed-releases/README.md)
- [ ] `shasum -a 256` every archive
- [ ] Create `sources.lock` with SNOMED_INT, SNOMED_IN/CDCI entries: version,
      sha256, licence, `redistributable: false`
- [ ] **Commit `sources.lock`** — it holds versions and checksums, never content

**Done when** `sources.lock` is committed and every archive's checksum matches.

### P3 — Inspect the releases
**Owner** Engineering + terminologist · **0.5 day** · **Blocks** P4, P7

Run the recipe in [`snomed-releases/README.md`](../../snomed-releases/README.md#inspection-recipe--answers-three-open-questions-in-20-minutes).
It answers by direct examination:

| Check | Decides |
|---|---|
| UNII map refset present? ([Q22](10-open-questions.md#q22)) | Whether RxNorm is structurally necessary or merely useful |
| Semantic-tag distribution in CDCI | Confirms the product/substance split empirically |
| Module dependency pairing | Whether the two packages actually load together |
| `Is modification of` edge count | Size of the salt/ester/prodrug classification job |
| `Has active ingredient` count in CDCI | Whether product decomposition works as designed |

- [ ] All five run, results recorded in the Phase 0 report
- [ ] Any surprise raised before P7, not after

**Done when** Q22 is answered and the relationship counts are written down.

### P4 — Evaluate CSNOServ / BHTS for ECL
**Owner** Terminologist + engineering · **0.5 day** · **Blocks** the Phase 1
tooling decision and a hardware request

[Q27](10-open-questions.md#q27). Check list in
[12 §A5](12-terminology-tooling.md#a5-what-to-check-before-committing-phase-0-half-a-day).

- [ ] Does `nrces.in/bhts/api/v1/csnoserv/` support ECL — FHIR
      `ValueSet/$expand` with an ECL filter, or natively? **Test the API, not the
      browser**; `/bhts/browser/` is CSNOFinder and works regardless
- [ ] Which FHIR terminology operations are available
- [ ] Which extension modules are loaded, at which release
- [ ] Rate limits / bulk suitability
- [ ] Can CSNOServ be deployed locally, offline?

**Done when** the Snowstorm question is settled. A yes on ECL removes 2.0
eng-weeks and the 16 GB build machine.

### P5 — Acquire DDInter and verify its actual contents
**Owner** Engineering + clinical pharmacology · **1 day** · **On the critical
path**

[Q3](10-open-questions.md#q3) — the scope question. **This step has grown teeth:
an initial look at the download page shows a likely serious coverage gap.**

**First, check you are on the right site.** DDInter 2.0 is at
**`ddinter2.scbdd.com`**; `ddinter.scbdd.com` is version 1.0. The 8-file
download set described below appears to be the 1.0 set. Everything under
[Q31](10-open-questions.md#q31) may look different on the 2.0 site — check there
before drawing any conclusion.

- [ ] **ATC first-level completeness** ([Q31](10-open-questions.md#q31)). The
      observed download set has 8 files — A, B, D, H, L, P, R, V — and is
      missing **C, G, J, M, N, S**. Confirm against the 2.0 site whether that
      holds. It matters more than the total record count, because the missing
      classes are where DDI risk concentrates:

      | Missing | Class | Why it matters |
      |---|---|---|
      | **C** | Cardiovascular | warfarin partners, digoxin, amiodarone, statins |
      | **J** | Anti-infectives | macrolides, azoles, rifampicin — the classic CYP perpetrators |
      | **N** | Nervous system | antidepressants, antipsychotics, antiepileptics, opioids — probably the largest single DDI category |
      | **M** | Musculoskeletal | NSAIDs, very high volume in India |
      | **G** | Genito-urinary / sex hormones | |
      | **S** | Sensory organs | the ophthalmic route cases ([11 C11](11-challenges-to-the-brief.md#c11)) |

- [ ] **Work out what is actually lost.** Each file holds interactions
      *involving* that class, so a pair survives if **either** partner is in a
      present class. The loss is pairs where **both** partners are in missing
      classes — azole × statin, SSRI × tramadol, phenytoin × carbamazepine,
      amiodarone × haloperidol. That is a smaller share of all pairs than the
      6-of-14 ratio suggests, and a **much larger** share of the severe ones.
- [ ] **Does the bulk CSV carry mechanism and management text, or only pair +
      severity?** ([Q32](10-open-questions.md#q32)) If only severity, findings
      have no actionable advice — "major interaction" with no "separate doses by
      4 hours, monitor TSH". That degrades the product independently of coverage
      and needs a decision, not a workaround.
- [ ] **Deduplicate across files.** An A × B interaction appears in both the A
      and B files. Union then dedup; report non-exact duplicates (same pair,
      different severity) rather than silently picking one.
- [ ] **Actual** drug and pair counts recorded (the brief's ~2,310 / ~302,000 are
      hypotheses to verify, not facts)
- [ ] Drug–food, drug–disease, therapeutic duplication present? Expectation:
      **drug–drug only**. If so, descope **in writing** with stakeholder
      sign-off; exact-moiety duplication remains detectable structurally at
      near-zero cost, labelled as structurally derived
- [ ] DrugBank cross-references published? ([Q23](10-open-questions.md#q23))

**Done when** the ATC coverage, the field set, and the true record counts are
known — and P7's sample is drawn with the missing classes explicitly in mind.

### P6 — Acquire RxNorm and the public-domain sources
**Owner** Engineering · **0.5 day** · Parallel

- [ ] UMLS/UTS account; RxNorm full monthly release
- [ ] Confirm `DRUGBANK` present as a source vocabulary ([Q23](10-open-questions.md#q23))
- [ ] UNII/GSRS, WHO ATC, openFDA, ONCHigh
- [ ] All entered in `sources.lock` with per-source `redistributable` flags — the
      build gate reads these, so a wrong flag is a compliance failure

### P7 — Coverage census · **THE GATE**
**Owner** Clinical pharmacology + engineering · **1 week** · Needs P3 and P5

Stratified sample of **150 molecules** from the AIIMS formulary:

| Stratum | n | Why |
|---|---|---|
| High-volume | 50 | What is actually prescribed |
| Random | 50 | Unbiased estimate |
| **NTI / QT / CYP-perpetrator** | 50 | Where missed interactions cause harm |

For each: present in DDInter? SNOMED substance? UNII? ATC? CDCI?

**Pass conditions:**

- [ ] ≥ 85% DDInter coverage of the high-volume stratum
- [ ] ≥ 70% DDInter coverage of the NTI/QT stratum

**Stop condition:** below 70% on NTI/QT, **do not build a general DDI checker**.
Re-scope to a curated high-priority-list service (ONCHigh + institution-authored
rules) — smaller, better-defined, still clinically valuable.

**Stratify against the P5 finding.** If the ATC gap in
[Q31](10-open-questions.md#q31) is real, the NTI/QT stratum will be
disproportionately affected — most of those molecules sit in C, J and N — and
this stop condition is the one that will bind. Draw the sample so that the
census *measures* the gap rather than accidentally avoiding it: report coverage
per ATC first level, not only in aggregate.

Also measure, while the sample is in hand: how many DDInter-missing molecules
DrugBank would actually add ([11 C7](11-challenges-to-the-brief.md#c7)). Expect
the increment to be small, because DDInter is largely DrugBank-derived — but if
the gap is whole ATC classes rather than scattered molecules, that expectation
may not hold, and it is worth re-testing rather than assuming.

### P8 — Get the AIIMS drug master
**Owner** Pharmacy + clinical informatics · **1 week elapsed, 0.5 day work**

[Q24](10-open-questions.md#q24). It is a spreadsheet, and looking at it collapses
most of the remaining uncertainty in the plan.

- [ ] Obtained
- [ ] Row count; FDC share; combi-pack share
- [ ] What coding it already carries, at what level
- [ ] Does it record route and dose form? (The request model needs them —
      [13 §2.1](13-hmis-neutral-integration.md#21-route-and-dose-form-must-be-inputs-not-inferences))

### P9 — Legal and licensing clearances
**Owner** Institutional legal · **2–3 weeks elapsed** · Start day 1

- [ ] [Q5](10-open-questions.md#q5) CredibleMeds redistribution — assume **no**,
      build the overlay regardless
- [ ] [Q6](10-open-questions.md#q6) ONCHigh licence status as redistributable data
- [ ] [Q7](10-open-questions.md#q7) NonCommercial and a fee-levying private ward;
      and the answer for a private hospital that asks
- [ ] [Q29](10-open-questions.md#q29) **ask NRCeS**: does a `codes-only`
      deployment need its own site licence? Decides whether a primary-care
      rollout needs zero registrations or one per site
- [ ] Licence memo signed, naming what may and may not be redistributed
- [ ] Affiliate License Agreement (2023) PDF captured verbatim into
      `licence.retrieved_text` with its retrieval date

### P10 — Governance and staffing
**Owner** Clinical informatics lead · **2–3 weeks elapsed** · Start day 1

The items most likely to be deferred, and the ones that later stall Phase 3.

- [ ] [Q10](10-open-questions.md#q10) **Named** makers and checkers, with
      committed weekly hours endorsed by department heads
- [ ] [Q13](10-open-questions.md#q13) What counts as a "department", and how many
- [ ] [Q15](10-open-questions.md#q15) Owner of the interruptive tier — ideally the
      existing DTC
- [ ] [Q16](10-open-questions.md#q16) Agreed interruptive alert-rate threshold,
      **pre-registered** before shadow mode
- [ ] [Q30](10-open-questions.md#q30) Owner of the annual Declaration of Use, and
      the deployment register started
- [ ] [Q17](10-open-questions.md#q17) IEC view on shadow-mode data collection

### P11 — Phase 0 exit review
**Owner** Project lead · **0.5 day**

- [ ] Coverage census passes, or the project is re-scoped
- [ ] Licence memo signed
- [ ] `sources.lock` committed; every source reproducible from checksum
- [ ] Q2, Q3, Q4, Q22, Q23, Q24, Q27 answered and recorded
- [ ] Phase 1 tooling decision made (CSNOServ vs Snowstorm)
- [ ] Go/no-go recorded, with the reasoning

---

## 3. Schedule

| | Week 1 | Week 2 | Week 3 |
|---|---|---|---|
| Critical path | P1 → P2 → P3 | P5, P7 starts | P7 completes, P11 |
| Parallel | P4, P6 | P8 | |
| Elapsed-time items — **start day 1** | P9, P10 → | → | → |

P9 and P10 involve other people's calendars and are the usual reason a 3-week
Phase 0 becomes 6. They have no technical dependency on anything above, so
there is no reason not to start both on the first day.

## 4. What Phase 0 produces

| Artifact | Committed? |
|---|---|
| `sources.lock` — versions, checksums, licence flags | **Yes** |
| Phase 0 report — census results, inspection findings, answered questions | **Yes** (`docs/phase-0/`) |
| Licence memo | No — institutional record |
| Deployment register, started | No — institutional record |
| Release archives | **Never** — gitignored, licensing control |
| Go/no-go decision | **Yes**, in the report |

## 5. Effort

| | Person-days |
|---|---|
| Engineering | 4 |
| Clinical pharmacology | 5 |
| Terminologist | 1 |
| Pharmacy | 1 |
| Project lead / legal / governance | 3 |
| **Total** | **~14 person-days over 3 weeks** |

Consistent with the 3 eng-weeks + 1 clinical-week in [07](07-effort.md); the
split shifts slightly toward clinical because the coverage census is the gate and
it is clinical work.
