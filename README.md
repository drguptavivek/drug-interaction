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
| 12 | [Terminology tooling](docs/plan/12-terminology-tooling.md) | Terminology server (build-time yes, runtime never); RxNorm's expanded role |
| 13 | [HMIS-neutral integration](docs/plan/13-hmis-neutral-integration.md) | What changes when SCTIDs live in the HMIS and no adapter is in scope |
| **14** | **[Phase 0 runbook](docs/plan/14-phase-0-runbook.md)** | **Start here to do the work** — ordered steps, owners, the go/no-go gate |
| 15 | [The four content types](docs/plan/15-content-types.md) | DDI, duplication, drug–food, drug–disease; the mechanism taxonomy and what it derives |

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
   coverage.** Now partly observed rather than hypothetical: the bulk download
   appears to ship only 8 of the 14 ATC first levels, missing cardiovascular,
   anti-infectives, nervous system, musculoskeletal, genito-urinary and sensory
   organs — the classes where DDI risk concentrates. Verify against
   `ddinter2.scbdd.com` (the 2.0 site) as the very first substantive Phase 0
   task; it may be a re-scope trigger.
7. **Run a terminology server (Snowstorm) for curation and build; never at
   runtime.** ECL and proper description search are worth the operational cost.
   The build consumes a checksummed ECL expansion cache, not a live server, so
   reproducibility survives. `ddid` has nowhere to configure one, deliberately.
8. **RxNorm deserves a bigger role than "interoperability".** Its structural path
   (`DDInter → DrugBank → RXCUI → SCTID`) needs no string matching, its `IN`/`PIN`
   split is an independent check on the riskiest decision in the design, and it
   may be the only way to derive the SNOMED-side UNII anchor. Its *brand* layer,
   by contrast, should be prohibited outright.
9. **HMIS neutrality moves the integration risk rather than removing it.** No
   adapter in scope, no vendor dependency on the critical path — but the drug
   master coding exercise it creates can produce confidently wrong findings that
   nobody downstream can see. A coding QA report, a `/v1/resolve` pre-flight, and
   two *mandatory* conformance fixtures are the replacements, and they are
   load-bearing.
10. **An FDC is one prescribed item with several ingredient codes**, not several
    items. `coding[]` (alternative codings of one product) and `ingredients[]`
    (composition) are separate fields; conflating them silently drops FDC
    components.

## Scale of the work

| | Person-weeks |
|---|---|
| Engineering | 39–46.5 |
| Clinical curation and departmental liaison | 7–9.3 |
| Pharmacy, drug master SCTID coding | 1.5–2.5 |
| **Calendar, 2 engineers + part-time clinical panel** | **~7 months** |

Full breakdown in [07](docs/plan/07-effort.md).
