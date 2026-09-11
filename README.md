# Indian DDI Knowledge Service — Implementation Plan

A drug–drug interaction (DDI) knowledge service for Indian healthcare, exposed as
an API callable from HMIS/EMR systems. Primary deployment target is an
AIIMS-level tertiary hospital; the same artifact is deployable to district
hospitals, CHCs and PHCs with different formularies and no reliable
connectivity.

**This repository currently contains a plan, not an implementation.** No
production code has been written. The documents below are the deliverable.

## Read in this order

| # | Document | What it answers |
|---|----------|-----------------|
| 00 | [Executive summary](docs/plan/00-executive-summary.md) | Shape of the system, headline numbers, what I think is wrong with the brief |
| 01 | [Phase breakdown](docs/plan/01-phases.md) | Entry/exit criteria per phase, sequencing, calendar |
| 02 | [Data model](docs/plan/02-data-model.md) | DDL for hub, spokes, salt/product, proposal/review, releases |
| 03 | [Candidate ranking](docs/plan/03-candidate-ranking.md) | Retrieval, scoring, evidence shown to maker and checker |
| 04 | [ETL pipeline](docs/plan/04-etl-pipeline.md) | Ingestion, normalization, artifact build, packaging |
| 05 | [Go service](docs/plan/05-go-service.md) | Packages, in-memory index, startup validation, overlays |
| 06 | [API contracts](docs/plan/06-api-contracts.md) | REST + CDS Hooks sketches, outcome taxonomy |
| 07 | [Effort estimate](docs/plan/07-effort.md) | Person-weeks, engineering vs clinical curation |
| 08 | [Licensing analysis](docs/plan/08-licensing.md) | What may be published, SNOMED India licence, ShareAlike |
| 09 | [Risk register](docs/plan/09-risks.md) | Risks, mitigations, owners, triggers |
| 10 | [Open questions](docs/plan/10-open-questions.md) | Decisions required before build starts |
| 11 | [Challenges to the brief](docs/plan/11-challenges-to-the-brief.md) | Constraints I believe are mistakes, with reasons |

## Headline conclusions

1. **Do not embed the knowledge base in the Go binary.** The CC BY-NC-SA 4.0
   ShareAlike obligation attaches to the compiled artifact if you do. Ship a
   signed sidecar file. See [08](docs/plan/08-licensing.md) and
   [11](docs/plan/11-challenges-to-the-brief.md#c1).
2. **Do not ship CredibleMeds data.** Its terms do not permit redistribution.
   Load it per-site as a licensee-supplied overlay.
3. **Do not derive salt→moiety collapse automatically from
   `Is modification of`.** That relationship also covers esters and prodrugs,
   whose interaction profiles differ from the parent moiety. Use it to
   *propose*, then adjudicate through the same maker-checker workflow used
   everywhere else in the design.
4. **Do not refuse to start on terminology drift.** Fail the *release*, not the
   *process*. A CDS service that will not boot removes interaction checking
   from the hospital entirely.
5. **Route, not just salt, gates interaction applicability.** Topical, ophthalmic
   and inhaled products are the general case of the iron oral-vs-parenteral
   exception already in the brief.
6. **The dominant risk is neither software nor terminology — it is DDInter
   coverage of the Indian formulary.** Measure it in Phase 0 before committing
   to the rest.

## Scale of the work

| | Person-weeks |
|---|---|
| Engineering | 34–42 |
| Clinical curation and departmental liaison | 7–10 |
| **Calendar, 2 engineers + part-time clinical panel** | **~7 months** |

Full breakdown in [07](docs/plan/07-effort.md).
