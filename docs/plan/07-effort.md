# 07 — Effort Estimate

Person-weeks, split by discipline. A person-week is 5 days at ~6 productive
hours. Clinical time is costed at the *hours actually spent adjudicating*, not
at the calendar time it takes to extract those hours from a consultant's week —
which is roughly 4× longer and is a scheduling problem, not an effort problem.

## 1. Engineering

| Phase | Workstream | Person-weeks | Confidence |
|---|---|---:|---|
| 0 | Source acquisition, checksums, licence dossier | 1.0 | High |
| 0 | Coverage census (tooling + analysis) | 1.5 | High |
| 0 | Artifact/licensing distribution design | 0.5 | Medium |
| 1 | SNOMED RF2 loader, substance graph, closures (reduced: ECL replaces hand-rolled SQL) | 1.0 | High |
| 1 | Snowstorm standup (Intl + India extension), ECL expansion cache, CI wiring | 2.0 | Medium |
| 1 | RxNorm ingest (`SAB=RXNORM` filter, IN/PIN graph, DrugBank/UNII cross-refs) | 0.5 | High |
| 1 | DDInter / UNII / ATC / openFDA / ONCHigh normalization | 1.5 | Medium |
| 1 | Anchor computation and agreement views | 1.0 | Medium |
| 1 | Reproducible-build harness | 0.5 | High |
| 2 | PostgreSQL schema, triggers, projection rebuild | 1.5 | High |
| 2 | Candidate ranking service (8 retrievers + scorer, incl. the RxNorm structural path) | 3.5 | **Low** |
| 2 | Curation API (Go) + Keycloak roles | 1.5 | High |
| 2 | Svelte curation UI (queue, evidence, review, release) | 2.5 | Medium |
| 2 | Release assembly + diff report | 0.5 | High |
| 3 | Departmental intake tooling, dedup, normalization (N1–N7) | 2.0 | Medium |
| 3 | FDC decomposition tooling | 1.0 | **Low** |
| 4 | Artifact format: writer, reader, CRC, signing | 2.0 | Medium |
| 4 | `kbindex` CSR + resolver + predicate evaluator | 2.0 | High |
| 4 | REST surface + OpenAPI + contract tests | 1.5 | High |
| 4 | Offline auth, config, overlay loader, startup validation | 1.5 | Medium |
| 4 | `ddictl` (verify/inspect/diff/dump/bench) | 1.0 | High |
| 4 | Observability, audit log | 0.5 | High |
| 4 | Performance work to hit p99 ≤ 5 ms / 0 allocs | 0.5 | Medium |
| 5 | CDS Hooks discovery + both hooks, IG conformance | 2.0 | Medium |
| 5 | HMIS adapter/shim | 2.0 | **Low** |
| 5 | Shadow-mode instrumentation | 1.0 | Medium |
| 6 | Tiering engine + overlay authoring tooling | 1.5 | Medium |
| 6 | Alert-volume replay harness | 1.0 | Medium |
| 6 | Primary-care overlay build | 1.0 | Medium |
| 6 | Clinical safety case support (hazard log tooling, traceability) | 0.5 | Medium |
| 7 | Pilot support, deployment docs, runbook, handover | 2.5 | Medium |
| all | CI, packaging, release engineering, security review | 2.0 | Medium |
| | **Subtotal** | **44.0** | |

**Low-confidence items, and why:**

- *Candidate ranking (3.5)* — eight retrievers over heterogeneous sources, plus
  a calibration loop. Could be 2.0 if SNOMED synonym coverage of Indian
  molecule names is good; could be 5.0 if it is poor and N6 orthographic
  handling turns into a research problem.
- *FDC decomposition (1.0)* — entirely dependent on whether CDC-India supplies
  structured composition. If it supplies only names, this becomes a parsing
  project: 3.0+.
- *HMIS adapter (2.0)* — a placeholder until the real interface is known. Could
  be 0.5 (clean REST) or 6.0 (a proprietary interface requiring vendor
  involvement and a change request). **This is the single largest scheduling
  unknown in the plan** and is [Q1](10-open-questions.md#q1).

**Engineering range: 35 (optimistic) – 44 (planning) – 60 (pessimistic).**

The +2.0 against the first draft is the terminology-server and RxNorm decisions
in [12](12-terminology-tooling.md): +2.0 for Snowstorm and the ECL cache, +1.0
for RxNorm, −1.0 saved on hand-written closure and search SQL. Both buy
correctness on the parts of the design most likely to fail quietly.

The optimistic case assumes CDC-India ships structured composition, the HMIS
speaks REST, and SNOMED synonym coverage is good. Plan to 42.

## 2. Clinical curation

Derived from the throughput assumptions in [03 §9](03-candidate-ranking.md#9-throughput-assumptions),
calibrated in Phase 2 against 50 real molecules.

| Activity | Basis | Hours | Person-weeks |
|---|---|---:|---:|
| Departmental data collection (25 departments × 2 h liaison) | 1,250 rows | 50 | 1.7 |
| Validation-set collection (same sessions, incremental) | ≥ 150 pairs | 15 | 0.5 |
| Suppression-list collection (same sessions) | | 10 | 0.3 |
| **Maker adjudication** — 650 molecules, band mix A/B/C = 65/25/10 | | 52 | 1.7 |
| **Checker review** — same 650 | | 25 | 0.8 |
| FDC decomposition review (~250 FDC products) | 6 min each | 25 | 0.8 |
| Salt collapse decisions (~2,000 salts, batched; `unknown`/ester/prodrug rows dominate) | | 20 | 0.7 |
| Exception rule authoring and review (~40 rules) | 30 min each | 20 | 0.7 |
| Alert tier cutting (Phase 6, iterative over shadow data) | 3 rounds | 24 | 0.8 |
| Blind re-audit (10% sample, third clinician) | 65 molecules | 12 | 0.4 |
| Clinical safety case / hazard log | | 16 | 0.5 |
| Pilot review meetings (Phase 7) | | 12 | 0.4 |
| | **Subtotal** | **281 h** | **9.3** |

**Clinical range: 7 (if band-A share reaches 75% and FDC handling is clean) –
9.3 (planning) – 15 (if DDInter coverage is poor and the hard queue grows).**

Note the shape of this: **the actual mapping adjudication is only ~27% of
clinical time.** Departmental liaison, exception authoring and alert tiering
together are larger. Plans for this kind of system routinely budget only for the
mapping and then stall in Phase 6 — the alert tier is where the clinical time
really goes, and it is the part that determines whether the system is used.

## 3. Roles and calendar

| Role | Allocation |
|---|---|
| Backend engineer (Go + Python) | 1.0 FTE for 7 months |
| Build machine for Snowstorm | 16 GB RAM, ~50 GB disk — request in Phase 0 |
| Full-stack engineer (Svelte + Go, ETL support) | 0.8 FTE for 5 months |
| Clinical pharmacologist (checker, tier owner) | 0.2 FTE for 5 months |
| Residents (makers, 2–3 rotating) | ~0.3 FTE aggregate for 3 months |
| Terminologist / SNOMED-experienced analyst | 0.2 FTE for 3 months, concentrated on the hard queue |
| Project lead / clinical informatics | 0.2 FTE throughout |

**Calendar: ~7 months to Phase 7 exit**, with the critical path running
Phase 0 → 1 → 2 → 3 → 6 → 7. Phase 4 is off the critical path only if a second
engineer is available from month 3; with a single engineer the calendar extends
to ~10 months.

A terminologist is listed separately because the hard queue (band C, ~10% of
molecules and a much larger share of total difficulty) is genuinely specialist
work. Without one, that 10% either stalls or gets adjudicated badly, and it is
disproportionately the unusual molecules where errors matter.

## 4. What is not in the estimate

Stated explicitly so it is not discovered later:

| Excluded | Why | Rough size if added |
|---|---|---|
| Drug–disease and drug–food interactions | Dependent on [C3](11-challenges-to-the-brief.md#c3); probably not in DDInter | +6–10 eng-weeks, +4 clinical-weeks, **plus a source** |
| Dose- and renal-function-dependent rules | Not phase 1; contract fields reserved | +8 eng-weeks, +6 clinical-weeks |
| Allergy/cross-sensitivity checking | Different knowledge domain entirely | separate project |
| Duplicate therapy beyond exact moiety match | Needs a class hierarchy decision | +3 eng-weeks |
| Pharmacogenomic interactions | Out of scope | — |
| Multi-site rollout beyond the second pilot site | Operations, not build | ~1 week per site |
| CDSCO medical-device regulatory submission | See [09](09-risks.md) R11 | unknown; legal-led |
| Ongoing maintenance after handover | Quarterly re-census | ~0.5 person-week/quarter + 4 clinical-hours/quarter |
