# HANDOFF

State of the work as of **12 September 2026**, at the end of the planning
session. Written for whoever picks this up next — most likely a local Claude Code
session on the developer's laptop, where the licensed source files live.

`CLAUDE.md` is the standing context and the eleven hard rules. **This file is the
"where we got to and what to do next".**

---

## 1. Status in one paragraph

The plan is complete and internally consistent: seventeen documents covering
phases, data model, candidate ranking, ETL, service design, API contracts,
effort, licensing, risks, open questions, and a Phase 0 runbook. **No production
code exists and none should be written until Phase 0's coverage census passes.**
The SNOMED Affiliate Licence is held, MLDS access is in place, and the SNOMED and
DDInter release files are downloaded on the laptop. Four questions can be
answered this week by inspecting those files, and one of them (Q22) changes the
ranker's weights.

## 2. What is in the repository

```
README.md                 index + headline conclusions
CLAUDE.md                 standing context; eleven hard rules
HANDOFF.md                this file
docs/plan/00–16           the plan
snomed-releases/          tracked folder, gitignored payload + inspection recipe
ddinter/                  tracked folder, gitignored payload + inspection recipe
.gitignore                a compliance control — read the comment at the top
```

Nothing licensed is committed, and that is deliberate: the repository is
**public**, so the `.gitignore` is a licence-compliance control rather than
housekeeping. Sources are pinned by SHA-256 in `sources.lock` (not yet created —
Phase 0 step P2) and fetched locally.

## 3. What changed this session, and why

Eight commits. The decisions, in the order they were taken:

| Decision | Effect |
|---|---|
| **KB ships as a signed sidecar, never `go:embed`** | Embedding attaches the CC BY-NC-SA ShareAlike obligation to the compiled binary |
| **CredibleMeds never shipped** | Redistribution almost certainly not permitted; sites supply an overlay under their own registration |
| **Salt→moiety collapse machine-proposed, human-approved** | `Is modification of` also covers esters, prodrugs and complexes |
| **Terminology drift fails the release, not the process** | A CDS service that won't boot removes interaction checking silently |
| **Route is the general applicability gate** | Not an iron-specific exception |
| **HMIS-neutral; SCTIDs supplied by the caller** | Removed the adapter — previously the largest estimate variance |
| **`coding[]` and `ingredients[]` are separate fields** | An FDC is one prescribed item with several ingredient codes |
| **CSNOServ is Snowstorm** | No separate Snowstorm standup; BHTS hosted for curation |
| **RxNorm and openFDA dropped; GSRS is the substance authority** | GSRS issues UNIIs and its `ACTIVE MOIETY` relationship is a better collapse cross-check |

### One correction worth reading before re-treading it

**C3 is withdrawn. I was wrong.** I challenged the brief's claim that DDInter 2.0
carries drug–food, drug–disease and therapeutic duplication data, and called it
"blocking (scope)". It carries all three — 857 DFI, 8,359 DDSI, 6,033
duplication — and the brief's DDI figures (302,516 records over 2,310 drugs) were
exact rather than approximate.

I inferred from the general shape of the field rather than from DDInter's own
documentation, and stated it far too strongly for something unverified. The
lesson is in the repo's style rule and is worth keeping: **where a source fact is
uncertain, route it to a Phase 0 check without also asserting the expected
answer.**

Scope is therefore *larger* than the drug–drug-only plan. See
`docs/plan/15-content-types.md`.

### The best thing found this session

DDInter annotates every interaction with a **mechanism category** — absorption,
distribution, metabolism, excretion, synergy, antagonism, others, unknown. That
was not in the brief and I would not have anticipated it.

Absorption-mechanism interactions are luminal, so the category *derives* the
route gate that was planned as hand-curated exceptions. `EXC_IRON_ROUTE` stops
being a special case and becomes one instance of a general rule. It removes
perhaps half the exception-curation work, and it tiers better than severity
alone.

## 4. Pick up here

Ordered. The runbook (`docs/plan/14-phase-0-runbook.md`) has owners and durations.

### This week, from files already on the laptop

1. **Q22 — is there a UNII map refset?** *Do this first.* Recipe in
   `snomed-releases/README.md`. With RxNorm dropped there is no fallback: absent
   the refset, both spokes derive UNII partly by name and feature F3 drops from
   weight 25 to 15, with ATC overlap and corroboration raised to compensate. It
   is a `grep`, and it changes the ranker.
2. **Q2 and the relationship counts** — same recipe. Confirms the
   product/substance split empirically and sizes the `Is modification of`
   classification job.
3. **Q31 / Q32 — DDInter.** Recipe in `ddinter/README.md`. Does the bulk download
   expose the full database (all 14 ATC first levels), and does the CSV carry the
   mechanism category and management text? The mechanism category is the field to
   look for first — see §3.
4. **GSRS export**, *including substance relationships*, not just the flat UNII
   code list. `ACTIVE MOIETY` is the whole point.

### Started today, because they have lead times

5. **Confirm CDCI access on MLDS.** The only remaining licensing item on the
   critical path. An International Edition download does not imply access to the
   national packages.
6. **Ask to see the AIIMS drug master.** It is a spreadsheet, and looking at it
   collapses most of the remaining uncertainty: row count, coding level, FDC and
   combi-pack share, whether it records route and dose form.
7. **Legal** (Q5, Q6, Q7, Q29) and **governance/staffing** (Q10, Q13, Q15, Q16,
   Q30). No technical dependency on anything above, and the usual reason a
   three-week Phase 0 becomes six.

### Then

8. `sources.lock` committed with every checksum and row count.
9. **The coverage census — the go/no-go gate.** 150 molecules, stratified,
   reported *per ATC first level* not in aggregate. **Stop condition: below 70%
   DDInter coverage of the NTI/QT stratum, re-scope** to a curated
   high-priority-list service rather than pushing on.

## 5. Open questions at a glance

| Closed | |
|---|---|
| Q2 | India extension supplies products; International supplies substances |
| Q3 | DDInter carries all four content types — C3 withdrawn |
| Q4 | CDCI is RF2 with real drug-model structure, via MLDS |
| Q23 | Withdrawn with RxNorm |
| Q28 | Affiliate Licence held, MLDS access in place |
| Q20/Q21 | Superseded — BHTS and CSNOServ are the national services |

| Live, highest value first | Why |
|---|---|
| **Q22** | No fallback now; changes ranker weights. A `grep`. |
| **Q31 / Q32** | Does the download expose everything, and does it carry the text |
| **Q24** | The drug master — most uncertainty collapsed per minute spent |
| **Q29** | Does a `codes-only` site deployment need its own licence? Decides whether primary-care rollout needs zero registrations or one per site |
| **Q27** | ECL endpoint shape — an hour, and the ECL client needs it |
| Q5, Q6, Q7 | Legal clearances |
| Q10, Q13, Q15, Q16, Q30 | Governance, staffing, alert-tier ownership, annual declaration |

Full text and defaults: `docs/plan/10-open-questions.md`.

## 6. Numbers

| | |
|---|---|
| Engineering | **37–44.5 person-weeks** (core build) |
| Clinical curation | 7–9.3 person-weeks |
| Pharmacy (drug master coding) | 1.5–2.5 person-weeks |
| Calendar | ~7 months with two engineers |
| DFI + duplication + renal/hepatic DDSI | +6.0 eng, ~+0.5 clinical on top |
| Full problem-list DDSI | +6.0 eng, +3.0 clinical — deferred, and I would keep it deferred |

The effort table's subtotal is the sum of its rows. It drifted 0.5 twice during
this session before being reconciled; keep it summed when rows change.

## 7. Two risks that decide whether this gets used

- **R1 — DDInter coverage of the Indian formulary (20).** The census gate exists
  for this. It is the one risk that cannot be designed around.
- **R2 — Alert fatigue (20).** Shadow mode before go-live, ≤ 2 interruptive
  alerts per 100 orders as a gate, and an override rate above 80% in pilot is a
  documented **stop condition** that halts rollout.

If the programme comes under schedule pressure, those two gates are the ones that
must not be waived. Everything else in the risk register is recoverable.

## 8. Things a future session should not quietly undo

The eleven hard rules in `CLAUDE.md`, and in particular the three most likely to
be "simplified" by someone with good intentions:

- **Embedding the KB in the binary** to get a true single-file deploy. It
  attaches ShareAlike to the compiled artifact.
- **Calling BHTS from `ddid`** for the startup terminology check, because the
  national server is right there. It would destroy the offline property. `ddid`
  has no terminology config key, deliberately.
- **Collapsing `partial` into `no_interactions_found`** because the API feels
  verbose. That distinction is the plan's main safety mechanism, and with an
  HMIS-neutral contract its rendering is now outside our control — which is why
  two conformance fixtures are mandatory rather than advisory.

If one of these looks wrong, argue it against the document that sets it out
rather than reversing it silently. Every one has its reasoning recorded.
