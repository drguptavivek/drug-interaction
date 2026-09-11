# 01 — Phase Breakdown

Eight phases. Phases 3 and 4 deliberately overlap: bulk clinical curation is
calendar-bound by clinician availability, and engineering should not idle behind
it.

```
Month   1      2      3      4      5      6      7
P0  ████
P1     ██████
P2        ████████████
P3                 ████████████████
P4                 ██████████████
P5                          ██████████
P6                                 ██████████
P7                                        ██████
```

---

## Phase 0 — Feasibility, licence clearance, coverage census

**Duration** 3 weeks · **Effort** 3 eng-weeks + 1 clinical-week

This phase exists to kill the project cheaply if the sources do not support it.

**Entry criteria**
- Named AIIMS executive sponsor and a named clinical pharmacology lead.
- Institutional legal contact identified for licence review.

**Work**
1. Acquire every source, record SHA-256, record the exact licence text as
   retrieved, with retrieval date. Do not rely on secondary descriptions.
2. Answer three factual questions that the brief currently assumes:
   - Does DDInter 2.0 actually contain drug–food, drug–disease and therapeutic
     duplication content, or only drug–drug? (See [C3](11-challenges-to-the-brief.md#c3).)
   - Do CredibleMeds terms permit redistribution in a derived artifact? (Expect: no.)
   - Does the SNOMED CT India edition contain substance concepts of its own, or
     does it extend only the product/dose-form space on top of the
     International Release substance hierarchy?
3. **Coverage census.** Take a stratified sample of 150 molecules from the AIIMS
   formulary (50 high-volume, 50 random, 50 narrow-therapeutic-index/QT/CYP
   perpetrators). For each, determine presence in: DDInter, SNOMED substance
   hierarchy, UNII, ATC. Report a coverage matrix.
4. Draft the artifact distribution model against the ShareAlike obligation
   (see [08](08-licensing.md)).

**Exit criteria**
- [ ] Licence memo signed by institutional legal, naming what may and may not be
      redistributed, with the CC1/CC2/CC7 decisions in [11](11-challenges-to-the-brief.md) resolved.
- [ ] **SNOMED International Affiliate Licence obtained via NRCeS** ([Q28](10-open-questions.md#q28)).
      Free — India is a Member country — but it gates Phase 2, because a
      non-Affiliate may not copy SCTIDs into a database, which is what the
      curation platform does.
- [ ] **[Q29](10-open-questions.md#q29) asked of NRCeS**: does a `codes-only` deployment need
      its own site licence? Decides whether a primary-care rollout needs zero
      registrations or one per site.
- [ ] Coverage census shows ≥ 85% DDInter coverage of the high-volume stratum
      and ≥ 70% of the NTI/QT stratum. Below that, the project changes shape
      (see "Stop conditions" below).
- [ ] All sources reproducible from a pinned manifest with checksums.
- [ ] Go/no-go decision recorded.

**Stop conditions** — if DDInter coverage of the NTI/QT stratum is below 70%, do
not proceed to build a general DDI checker. Re-scope to a curated
high-priority-list-only service (ONCHigh + institution-authored rules), which is
a smaller, better-defined and still clinically valuable system.

---

## Phase 1 — Source ingestion and terminology graph

**Duration** 4 weeks · **Effort** 5 eng-weeks

**Entry** Phase 0 exit met; sources on disk with checksums.

**Work**
- Python ETL stages `acquire` → `stage` → `normalize` (see [04](04-etl-pipeline.md)).
- Stand up **Snowstorm** with the International Release + India extension, for
  ECL, description search and the terminologist's browser. Build the ECL
  expansion cache and wire it into CI. See [12 Part A](12-terminology-tooling.md#part-a--terminology-server).
  Needs a 16 GB build machine — request it in Phase 0.
- Load SNOMED RF2 snapshot (International + India extension) into PostgreSQL:
  `concept`, `description`, `relationship`, `refset` tables, release-tagged.
- Materialise the substance sub-hierarchy (`105590001 | Substance |` descendants)
  and the `738774007 | Is modification of |` graph, including its transitive
  closure with cycle detection.
- Materialise product decomposition: `has active ingredient` (127489000),
  `has precise active ingredient` (762949000), `has basis of strength substance`
  (732943007), plus dose form and route.
- Normalize DDInter into `ddinter_drug` and `ddinter_interaction`.
- Load UNII/GSRS names, WHO ATC index, openFDA label extracts, ONCHigh list.
- Load RxNorm (`SAB=RXNORM` only): `IN`/`PIN`/`MIN` graph, SNOMEDCT_US atoms,
  DrugBank cross-references, UNII attributes. See [12 Part B](12-terminology-tooling.md#part-b--rxnorm).
- Build the anchor tables.

**Exit criteria**
- [ ] `make ingest` reproduces the staging DB bit-identically from the pinned
      manifest on a clean machine.
- [ ] Row-count assertions pass for every source and are recorded in the
      manifest, including DDInter's actual drug and interaction counts (the
      brief's ~2,310 / ~302,000 are treated as *assertions to verify*, not facts).
- [ ] `Is modification of` closure has no cycles; any found are reported, not
      silently broken.
- [ ] Anchor disagreement rate measured and reported.
- [ ] Every ECL expression used by the build is named, cached and checksummed;
      a cache miss fails CI.
- [ ] Whether the release ships a UNII map refset is determined, and the
      SNOMED-side UNII derivation path is confirmed to work by one route or the
      other ([Q22](10-open-questions.md#q22)).

---

## Phase 2 — Curation platform

**Duration** 6 weeks · **Effort** 9 eng-weeks (2 engineers)

**Entry** Phase 1 exit met.

**Work**
- PostgreSQL schema from [02](02-data-model.md), with append-only enforcement at
  the database level (revoked `UPDATE`/`DELETE`, plus triggers).
- Maker-checker enforcement: `reviewed_by <> proposed_by` as a database trigger,
  not application code, not the UI. A server-side integration test that attempts
  self-approval via the API and asserts rejection is a required deliverable.
- Candidate ranking service ([03](03-candidate-ranking.md)).
- Svelte curation UI: work queue, evidence panel, propose, review, exception
  flagging, release assembly, diff report.
- Keycloak integration with realm roles `ddi-maker`, `ddi-checker`, `ddi-release`.
  A single user may hold both maker and checker roles; the trigger still prevents
  self-approval on any individual proposal.

**Exit criteria**
- [ ] Integration test suite proves: self-approval rejected; `UPDATE` on
      `proposal` rejected; projection rebuild from the append-only log is
      deterministic and idempotent.
- [ ] 50-molecule pilot curated end-to-end by two real clinicians, timed
      (this calibrates the Phase 3 estimate in [07](07-effort.md)).
- [ ] Release `r0` assembled with a human-readable diff report.
- [ ] Median maker adjudication time ≤ 5 minutes/molecule measured on the pilot.

---

## Phase 3 — Departmental collection and bulk curation

**Duration** 8 weeks calendar · **Effort** 3 eng-weeks + 6–9 clinical-weeks

**Entry** Phase 2 exit met; departmental liaison appointed per department.

**Work**
- Collect from each department: top 50 salts; **plus** the department's known
  worked-around interaction pairs (validation set); **plus** alerts they would
  want suppressed. The latter two are as valuable as the first and are usually
  the ones that get dropped when time runs short — collect them in the *same*
  form, at the same time.
- Deduplicate ~1,250 rows to the expected 500–650 molecules.
- Sequence curation by **department-count descending**, as the brief specifies —
  correct, because cross-specialty pairs are where real harm lives. Add a second
  sort key: NTI/QT/CYP-perpetrator status, so that a molecule appearing in one
  department but carrying high interaction risk (e.g. a single-department
  antiarrhythmic) is not left to the end.
- Curate. Flag exceptions. Adjudicate anchor disagreements.
- Decompose fixed-dose combinations — expect this to be 25–35% of Indian product
  rows and to be the slowest part.

**Exit criteria**
- [ ] ≥ 95% of molecules appearing in ≥ 3 departments approved.
- [ ] ≥ 90% of all phase-1 molecules approved.
- [ ] Validation set of ≥ 150 department-supplied pairs recorded with expected
      outcomes.
- [ ] Suppression request list recorded with rationale per entry.
- [ ] 10% blind re-audit shows ≥ 99% mapping precision.

---

## Phase 4 — Runtime service

**Duration** 6 weeks · **Effort** 9 eng-weeks · **Overlaps Phase 3**

**Entry** Phase 2 exit met; release `r0` exists to build against.

**Work**
- Artifact format and builder ([04](04-etl-pipeline.md#6-artifact-format)).
- Go service per [05](05-go-service.md): loader, CSR pair index, resolver,
  overlay engine, REST surface, offline JWT validation, audit log.
- `ddictl verify` / `inspect` / `diff` / `sign`.
- Startup validation as a **release gate** and a **runtime degradation**, not a
  boot failure ([C5](11-challenges-to-the-brief.md#c5)).

**Exit criteria**
- [ ] Contract tests pass against the API sketch in [06](06-api-contracts.md).
- [ ] p99 ≤ 5 ms for a 20-drug list; RSS ≤ 512 MB; cold start ≤ 3 s.
- [ ] Zero allocations on the hot lookup path (verified by benchmark).
- [ ] Service runs with the host's network interface down, end to end.
- [ ] Signature verification failure is fatal; terminology drift is not.

---

## Phase 5 — Integration contract, conformance and coding support

**Duration** 4 weeks · **Effort** 5.5 eng-weeks

**Entry** Phase 4 exit met. *No dependency on an HMIS test instance or a vendor
contact — that dependency was removed by the HMIS-neutral decision
([13](13-hmis-neutral-integration.md)).*

**Work**
- CDS Hooks `order-select` and `order-sign` per HL7 PDDI-CDS IG.
- **Conformance kit**: OpenAPI + CDS Hooks discovery documents, ~40 fixture
  request/response pairs covering every outcome-taxonomy branch, and
  `ddictl conformance --endpoint <url>` so any HMIS vendor can self-certify
  without us ([13 §4.4](13-hmis-neutral-integration.md#44-conformance-kit)).
- **Coding target set** export (`ddictl export --coding-set`) — the list of codes
  the KB understands, so the HMIS team codes against that rather than against the
  whole of SNOMED CT.
- **`ddictl qa-coding`** — batch quality report over a coded drug master.
- `IDX_HIST` historical associations, so stale HMIS codes resolve to their
  replacements rather than dead-ending.
- Shadow mode: the HMIS calls the service and logs findings without displaying
  them. Still needed, and still requires a cooperating HMIS — but now it is the
  institution's integration, tested against a published contract, not ours.

**Exit criteria**
- [ ] CDS Hooks discovery and both hooks pass the public CDS Hooks sandbox.
- [ ] Conformance kit published; all fixtures pass against our own service.
- [ ] The two **mandatory** fixtures pass against the AIIMS integration: the
      unresolved array is displayed, and `partial` is not rendered as
      "no interactions found" ([R28](09-risks.md)).
- [ ] `qa-coding` run over the full AIIMS drug master; unresolved rate and FDC
      arity mismatches reported and triaged.
- [ ] Shadow mode running for ≥ 2 weeks with logged alert volumes.

---

## Phase 6 — Alert tiering, overlays, clinical safety

**Duration** 5 weeks · **Effort** 5 eng-weeks + 2 clinical-weeks

**Entry** Shadow-mode data available from Phase 5.

**Work**
- Cut the interruptive tier from shadow-mode volumes: start from ONCHigh ∩
  DDInter-major, subtract department suppression requests, iterate until
  ≤ 2 interruptive alerts per 100 orders.
- Build the primary-care overlay: PHC/CHC formulary from the national Essential
  Medicines List and state EDLs; a much smaller interruptive set.
- Clinical safety case and hazard log, modelled on ISO 14971 + the UK
  DCB0129 structure (no Indian equivalent mandates this; it is cheap and it is
  what an ethics committee and any future CDSCO discussion will ask for).
- Validation run against the departmental validation set; report sensitivity and
  the false-negative list.

**Exit criteria**
- [ ] Interruptive rate ≤ 2 per 100 orders in shadow data.
- [ ] Recall ≥ 90% against the validation set; every miss individually explained.
- [ ] Hazard log signed off by the clinical pharmacology lead.
- [ ] Two overlays built and tested from one binary.

---

## Phase 7 — Pilot, release, handover

**Duration** 4 weeks · **Effort** 4 eng-weeks + 1 clinical-week

**Entry** Phase 6 exit met; institutional ethics clearance for the pilot.

**Work**
- Live pilot in 2–3 departments with interruptive alerts enabled.
- Measure override rate. **Stop condition: override rate > 80% halts rollout**
  and returns to Phase 6 tiering.
- Deployment documentation, overlay authoring guide, release runbook.
- Handover to an institutional owner with a named maintainer.

**Exit criteria**
- [ ] Override rate ≤ 50%.
- [ ] Release `r1` signed and tagged; reproducible from the manifest.
- [ ] A second site (one CHC or district hospital) running the same binary with
      its own overlay.
- [ ] Named maintainer, documented quarterly re-census procedure
      ([C6](11-challenges-to-the-brief.md#c6)).
- [ ] **Deployment register live**, with both pilot sites recorded: site type,
      KB version, build profile, workstation count, contact. It is the source for
      the annual Declaration of Use *and* the distribution list for a withdrawn
      release ([08 §3.6.5](08-licensing.md#365-annual-declaration-of-use--a-recurring-obligation-the-plan-had-missed)).
- [ ] **Named owner for the annual Declaration of Use**, due 15 January via
      NRCeS ([Q30](10-open-questions.md#q30)).
