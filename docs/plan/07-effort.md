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
| 1 | ECL expansion cache + CI wiring against BHTS/CSNOServ (no Snowstorm standup — CSNOServ *is* Snowstorm) | 0.5 | Medium |
| 1 | GSRS ingest (UNII, names/synonyms, `ACTIVE MOIETY` relationships) | 0.5 | High |
| 1 | DDInter / UNII / ATC / openFDA / ONCHigh normalization | 1.5 | Medium |
| 1 | Anchor computation and agreement views | 1.0 | Medium |
| 1 | Reproducible-build harness | 0.5 | High |
| 2 | PostgreSQL schema, triggers, projection rebuild | 1.5 | High |
| 2 | Candidate ranking service (6 retrievers + scorer) | 3.0 | **Low** |
| 2 | Curation API (Go) + Keycloak roles | 1.5 | High |
| 2 | Svelte curation UI (queue, evidence, review, release) | 2.5 | Medium |
| 2 | Release assembly + diff report | 0.5 | High |
| 3 | Departmental intake tooling, dedup, normalization (N1–N7) | 2.0 | Medium |
| 3 | FDC decomposition tooling | 1.0 | **Low** |
| 4 | Artifact format: writer, reader, CRC, signing | 2.0 | Medium |
| 4 | `kbindex` CSR + resolver + predicate evaluator | 2.0 | High |
| 4 | REST surface + OpenAPI + contract tests (incl. `ingredients[]`/`coding[]`, `local_id`, route) | 2.0 | High |
| 4 | Offline auth, config, overlay loader, startup validation | 1.5 | Medium |
| 4 | `ddictl` (verify/inspect/diff/dump/bench) | 1.0 | High |
| 4 | Observability, audit log | 0.5 | High |
| 4 | Performance work to hit p99 ≤ 5 ms / 0 allocs | 0.5 | Medium |
| 5 | CDS Hooks discovery + both hooks, IG conformance | 2.0 | Medium |
| 5 | Conformance kit (fixtures + `ddictl conformance`) | 1.0 | High |
| 5 | Coding target set export + `ddictl qa-coding` | 1.0 | High |
| 5 | Drug master coding view in the curation UI | 1.0 | Medium |
| 5 | `IDX_HIST` historical associations (ETL + artifact + resolver) | 0.5 | High |
| 5 | Shadow-mode instrumentation | 1.0 | Medium |
| 6 | Tiering engine + overlay authoring tooling | 1.5 | Medium |
| 6 | Alert-volume replay harness | 1.0 | Medium |
| 6 | Primary-care overlay build | 1.0 | Medium |
| 6 | Clinical safety case support (hazard log tooling, traceability) | 0.5 | Medium |
| 7 | Pilot support, deployment docs, runbook, handover | 2.5 | Medium |
| all | CI, packaging, release engineering, security review | 2.0 | Medium |
| | **Subtotal** | **44.5** | |

The subtotal is the sum of the rows above; keep it that way when rows change.

**Low-confidence items, and why:**

- *Candidate ranking (3.0)* — six retrievers over heterogeneous sources, plus
  a calibration loop. Could be 2.0 if SNOMED synonym coverage of Indian
  molecule names is good; could be 5.0 if it is poor and N6 orthographic
  handling turns into a research problem.
- *FDC decomposition (1.0)* — entirely dependent on whether CDC-India supplies
  structured composition. If it supplies only names, this becomes a parsing
  project: 3.0+.
The pessimistic case dropped from 60 to 52 because the HMIS adapter — previously
the single largest scheduling unknown, with a 0.5–6.0 range and no way to resolve
it before month 5 — was removed by the HMIS-neutral decision
([13 §8](13-hmis-neutral-integration.md#8-effort-delta)). The planning number rose
by 2.0 and the variance fell by roughly 8. That is the better trade.

**Engineering range: 37 (optimistic) – 44.5 (planning) – 50 (pessimistic).**

Of the movement since the first draft (42.5): +2.0 for the terminology-server and
RxNorm decisions in [12](12-terminology-tooling.md) (+2.0 Snowstorm and the ECL
cache, +1.0 RxNorm, −1.0 saved on hand-written closure and search SQL), and +2.0
for the HMIS-neutral decision in [13](13-hmis-neutral-integration.md). Both buy
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
| Build machine | **Not needed** if curation uses BHTS hosted and the build consumes the ECL expansion cache. Only a local CSNOServ deployment wants ~16 GB (Snowstorm means Elasticsearch) |
| Full-stack engineer (Svelte + Go, ETL support) | 0.8 FTE for 5 months |
| Clinical pharmacologist (checker, tier owner) | 0.2 FTE for 5 months |
| Residents (makers, 2–3 rotating) | ~0.3 FTE aggregate for 3 months |
| Terminologist / SNOMED-experienced analyst | 0.2 FTE for 3 months, concentrated on the hard queue |
| Pharmacy technician + pharmacist (drug master coding) | ~1.5–2.5 person-weeks, Phase 5 |
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
| Drug–food, drug–disease, therapeutic duplication | **Now in scope** — all three exist in DDInter ([15](15-content-types.md)). Not in the 46.5 figure above. | +6.0 eng-weeks, ~+0.5 clinical for duplication + DFI + renal/hepatic DDSI; full problem-list DDSI a further +6.0/+3.0 |
| Dose- and renal-function-dependent rules | Not phase 1; contract fields reserved | +8 eng-weeks, +6 clinical-weeks |
| Allergy/cross-sensitivity checking | Different knowledge domain entirely | separate project |
| ~~Duplicate therapy beyond exact moiety match~~ | DDInter supplies 96 pharmacological classes and 6,033 records — see [15 §3](15-content-types.md#3-therapeutic-duplication) | +2.0 eng-weeks |
| Pharmacogenomic interactions | Out of scope | — |
| Multi-site rollout beyond the second pilot site | Operations, not build | ~1 week per site |
| CDSCO medical-device regulatory submission | See [09](09-risks.md) R11 | unknown; legal-led |
| Ongoing maintenance after handover | Biannual terminology re-census on the SNOMED/CDCI release clock, lighter quarterly formulary check, annual Declaration of Use | ~0.5 person-week + 4 clinical-hours per SNOMED release; ~2 h/year for the declaration |
