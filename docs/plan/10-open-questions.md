# 10 — Open Questions

Each needs a decision before the phase named in "Blocks". Owner is a role, not a
person; assign real names at kickoff.

## Blocking Phase 0 exit

### Q1
**~~What protocol does the AIIMS HMIS actually speak?~~ — largely closed.**
Superseded by the HMIS-neutral decision ([13](13-hmis-neutral-integration.md)).
We publish a contract and a conformance kit; the calling system's protocol is
not our concern. What remains is narrower and is asked as Q24–Q26 below.

### Q24
**At what level will SCTIDs be added to the HMIS drug master — substance,
product, or CDC-India codes? And how many rows are there?**
Blocks: Phase 5 coding support tooling, and the pharmacy effort estimate.
Owner: Pharmacy + HMIS team.
See [13 §2](13-hmis-neutral-integration.md#2-the-crux-which-sctids-will-the-hmis-hold).
Expect mostly substance-level, since Indian brands will not exist as SNOMED
product concepts. The request model accommodates all three, so this does not
block the build — but it determines whether `route` and `dose_form` must come
from the HMIS (they must, under substance-level coding) and how large the coding
exercise is. **Ask to see the drug master in Phase 0**; it is a spreadsheet, and
looking at it removes most of the remaining uncertainty in the plan.

### Q25
**Who owns the drug master coding exercise, and does it go through
maker-checker?**
Blocks: Phase 5 governance.
Owner: Pharmacy + clinical governance.
Recommendation: yes to maker-checker, in the same platform. A mis-coded row
([R27](09-risks.md)) is higher-volume and lower-attention work than molecule
mapping, and its errors are invisible downstream. It deserves at least the same
control.

### Q26
**Will the AIIMS integration commit to passing the two mandatory conformance
fixtures** — displaying the unresolved array, and not rendering `partial` as
"no interactions found"?
Blocks: Phase 5 exit, and arguably go-live.
Owner: Clinical informatics lead + HMIS team.
This is the one integration requirement worth being inflexible about
([R28](09-risks.md)). Everything else in the taxonomy is advisory; these two are
the difference between "the service said nothing about this drug" and "the
service said this drug is safe".

### Q2
**~~Does the SNOMED CT India edition contain substance concepts?~~ — ANSWERED.**
**No — it supplies the product layer.** The CDCI package provides generic,
supplier and branded medicine concepts *for use alongside* the International
Release, which supplies substances. This is the good outcome: spoke A targets are
internationally stable concepts and the Indian dependency is confined to
products. Consequence retained — the startup check validates against **both**
release identifiers, not just the India edition as the brief states.

### Q3
**~~Does DDInter 2.0 actually contain drug–food, drug–disease and therapeutic
duplication data?~~ — ANSWERED: yes, all three.**
857 DFI records over 29 foods; 8,359 DDSI records over 472 diseases; 6,033
therapeutic duplication records over 317 combination drugs and 96 pharmacological
classes. Plus 302,516 DDI records over 2,310 drugs with 8,398 distinct mechanism
and management descriptions — the brief's figures were exact.

My challenge ([C3](11-challenges-to-the-brief.md#c3)) is
withdrawn. Scope is *larger* than the drug–drug-only plan, not smaller. Modelling
and cost for each content type: [15](15-content-types.md).

### Q4
**~~Is CDC-India obtainable in bulk with structured composition?~~ — ANSWERED.**
**Yes.** Common Drug Codes for India (CDCI) is an RF2 **Terminology Integrated
Package** distributed via MLDS, synchronous with the SNOMED CT International
Edition. It carries real SNOMED drug-model structure, so decomposition uses the
same `has active ingredient` relationships as any other product. The FDC parsing
project budgeted as a downside risk does not arise for covered products.

Two residual items, now design inputs rather than unknowns:
- **Combi packs are excluded** from CDCI. Such drug master rows must be
  decomposed into component products — a named check in `ddictl qa-coding`.
- **Branded (RCD) coverage is incremental** and national-programme-driven
  (e.g. +134 CD, +892 RCD in the August 2026 package). Expect a tertiary
  formulary to be partly RCD-coded and largely substance-coded, which is what the
  `coding[]` / `ingredients[]` request model already accommodates.

### Q5
**May CredibleMeds QTDrugs data be redistributed in a non-commercial
public-health artifact?**
Blocks: artifact contents, [08 §4](08-licensing.md#4-crediblemeds--do-not-ship-c2).
Owner: Institutional legal.
My position: assume no, build the site-supplied overlay. The overlay design is
correct either way, so this question does not block the build — only what the
shipped artifact contains.

### Q6
**What is the licence status of the ONCHigh high-priority list as redistributable
data (as opposed to a published paper)?**
Blocks: whether ONCHigh membership can be a shipped bitmap.
Owner: Institutional legal.
Fallback: the list is ~15 class pairs; re-derive the membership from the cited
primary literature as our own curated artifact, reviewed through maker-checker
like everything else. Slightly more clinical time, no licence exposure.

### Q7
**Is non-commercial use satisfied by a government tertiary hospital that levies
fees in private wards? And what is the answer for a private hospital that asks to
deploy?**
Blocks: deployment guide, and the answer given to the first private hospital that
asks.
Owner: Institutional legal.
Default: proceed for government and academic deployment as the brief scopes;
state the private-sector boundary plainly in the deployment guide.

### Q20
### Q21
**~~Ontoserver? Does NRCeS expose a hosted endpoint?~~ — SUPERSEDED.**
Both answered by the national services. **BHTS** — the Bharat Health Terminology
Service — is live at `nrces.in/bhts` with a FHIR-compliant API at
`/bhts/api/v1/csnoserv/`, and `nrces.in/bhts/browser/` is a **CSNOFinder**
search-and-browse UI. **CSNOServ** has been available since 2014 as a locally
deployable service, part of the **Apache-2.0** CSNOtk toolkit, carrying the
Indian extensions (AYUSH, CDCI) pre-integrated. Ontoserver is out of
consideration. See [12 §A3](12-terminology-tooling.md#a3-which-server--settled).

### Q27
**~~Does CSNOServ support ECL?~~ Yes — CSNOServ is Snowstorm.** *(narrowed)* What remains is
the practical endpoint shape: FHIR `ValueSet/$expand` with an ECL filter, or a
native ECL endpoint? The ECL client needs to know either way, and it is an hour
against the live endpoint, not an evaluation.
Owner: Engineering.
Also worth noting while there: rate limits and bulk suitability, since a
~5,000-row drug master may be resolved against it in a batch.

### Q28
**~~Is SNOMED International Affiliate registration in place, via MLDS?~~ — CLOSED.**
**Yes. The Affiliate Licence is held and MLDS access is in place.** The curation
platform is therefore licensed to copy SCTIDs into a database (CSNOServ
sub-licence clause 5), and Phase 2 is unblocked.

Two consequences that are now live rather than future:

- **CDCI access.** Confirm the Common Drug Codes for India national release is
  available under the existing registration — an International Edition download
  does not imply it. This is the only remaining licensing item on the critical
  path ([08 §3.6.6](08-licensing.md#366-obtaining-it--already-done-for-this-project)).
- **The annual Declaration of Use is already an obligation**, not a future one —
  see [Q30](#q30). Due 15 January.

### Q22
**Does the SNOMED release ship a UNII map reference set?** — *first-week item, now with no fallback.*
Blocks: the strength of the third-anchor check.
Owner: Terminologist. **A `grep` against files already on the laptop.**

With RxNorm dropped there is no `SCTID → RXCUI → UNII` fallback. Either the
refset exists and the SNOMED-side UNII anchor is structural, or it does not and
that side becomes a name match against GSRS substance names — at which point
**both** spokes derive UNII partly by name, and agreement is weaker evidence than
the design assumes.

| Outcome | Action |
|---|---|
| Refset present | No change. F3 keeps weight 25 |
| Refset absent | F3 drops to 15; raise ATC set-overlap (F4) and corroboration (F5); route more molecules to band B for human adjudication. **Do not keep treating UNII agreement as a strong signal** |

Recipe: [`snomed-releases/README.md`](../../snomed-releases/README.md#q22--is-there-a-unii-map-reference-set).
Check for an **ATC** map refset in the same listing while you are there
([16 §2](16-source-checklist.md#2-atc--the-licensing-trap-and-why-you-probably-do-not-need-to-buy-it)).

### Q23
**~~Does DDInter publish DrugBank cross-references, and does RxNorm carry
`DRUGBANK`?~~ — WITHDRAWN.**
Moot: RxNorm is out of scope, so the structural `DDInter → DrugBank → RXCUI →
SCTID` path it supported does not exist. GSRS replaces RxNorm's other two roles
([12 §B1](12-terminology-tooling.md#b1-why-rxnorm-was-proposed-and-why-gsrs-covers-it)).

### Q29
**Does a site that deploys only the `codes-only` KB need its own Affiliate
Licence?**
Blocks: the primary-care rollout model, not the build.
Owner: Institutional legal + NRCeS.
The 2023 Affiliate Licence update permits non-licensed systems to receive, store
and forward SNOMED codes and descriptions. A CHC or PHC running `ddid` with a
`codes-only` artifact holds SCTIDs and *our* curated names — no SNOMED
descriptions, no terminology content. If that counts as storing codes rather than
deploying SNOMED CT, a rollout to hundreds of primary-care sites needs **zero**
registrations; if it counts as deployment, it needs one per site.

Free either way in India, so this is administrative friction, not cost — but at
several hundred sites the difference decides whether the rollout is feasible.
**Ask NRCeS directly**; they are the authority and the question is squarely
theirs. See [08 §3.6.4](08-licensing.md#364-where-the-boundary-actually-falls).
Design is unaffected: both profiles already exist.

### Q30
**Who files the annual Declaration of Use, and is a deployment register being
kept from the first deployment?**
Blocks: Phase 7 handover; ongoing compliance. **Live now** — the licence is
already held ([Q28](#q28)), so the declaration obligation already exists,
independently of whether this project ships.
Owner: Institutional owner (same person as [Q19](#q19)).
Due **15 January** each year via NRCeS, reporting purpose of use, implementation
status, end users/sublicensees, and the **number of applications and
workstations**. The service has no telemetry by design and must not acquire any
for this, so the counts come from an administrative register that has to be
maintained from day one — reconstructing it later is far harder.
See [08 §3.6.5](08-licensing.md#365-annual-declaration-of-use--a-recurring-obligation-the-plan-had-missed).

### Q31
**Does the bulk download expose the full database, or a subset of it?** — *narrowed, and much less alarming.*
Blocks: how the KB is built, not whether the project is viable.
Owner: Engineering. **Half a day.**

**The content question is settled.** DDInter's own mechanism documentation works
through examples involving lovastatin and ketoconazole (C, J), rifampin and
verapamil (J, C), benzodiazepines and opioids (N), NSAIDs (M), beta-blockers (C),
lithium and methotrexate (N, L). Those classes are unambiguously *in the
database*. So the missing letters on the download page are a **packaging**
question, not a coverage hole in the resource.

What is still to check, on `ddinter2.scbdd.com/download/`:

- [ ] Are there files for **C, G, J, M, N, S**? The 8-file set observed at
      `ddinter.scbdd.com` is the 1.0 site.
- [ ] Are there separate downloads for **DFI, DDSI and therapeutic duplication**,
      or only DDI?
- [ ] Do the per-file record counts sum to the documented 302,516?

If the download really is a subset, the options are to ask the authors for the
full set (a reasonable request for a non-commercial academic deployment), or to
check whether per-drug page access is permitted under CC BY-NC-SA. Re-scoping is
a last resort now, not the expected outcome.

### Q32
**Does the DDInter bulk CSV carry the mechanism and management text, or only drug
pairs and a severity level?**
*(The text **exists** — 8,398 distinct mechanism and management descriptions for
DDIs, 430 for DFIs, 3,300 for DDSIs. The only question is whether the bulk CSV
ships it.)*
Blocks: the clinical usefulness of every finding; the `RULE` section of the
artifact ([04 §6](04-etl-pipeline.md#6-artifact-format)); the `mechanism` and
`management` response fields ([06 §1](06-api-contracts.md#1-post-v1interactionscheck)).
Owner: Engineering. **One hour — open a CSV.**

The DDInter *website* shows mechanism and management per interaction. Whether
the *bulk download* does is a different question. If it carries only
`{pair, severity}`, then a finding says "major interaction" with no advice —
no "separate administration by 4 hours, monitor TSH". Severity without management
is an alert that tells a clinician to worry and not what to do, which is a
direct contributor to override behaviour ([R2](09-risks.md)).

If absent, the options are: scrape per-interaction pages (slow, and check the
licence permits it), author management text institutionally for the interruptive
tier only (small, high-value, fully ours and publishable), or ship severity-only
findings and say so plainly in the UI. My preference is the second — the
interruptive tier is a few dozen rules, and that is exactly where advice matters.

## Blocking Phase 1

### Q8
**Should a public-domain-only "open tier" artifact be built** (openFDA + UNII +
ONCHigh, no DDInter), distributable without NC or ShareAlike constraints?
Owner: Project lead.
It is the only case where `go:embed` remains defensible. My view: not worth it in
phase 1 — such an artifact would have poor coverage and could mislead a site into
thinking it had a real DDI checker. Revisit after the coverage census.

### Q9
**Is `preferred_name` curated independently of the SNOMED FSN, accepting the
extra clinical effort, to preserve the `codes-only` distribution profile?**
Owner: Clinical pharmacology lead + legal.
Cost: a few seconds per molecule during curation. Benefit: the mapping table
becomes openly publishable to non-affiliates, which is the highest-reuse output
of the project. My recommendation: yes, and enforce it by not offering a
"copy FSN" button in the UI.

## Blocking Phase 2

### Q10
**Who are the makers and checkers, concretely, and what is their committed
weekly time?**
Owner: Clinical informatics lead.
Blocks: the Phase 3 calendar entirely. [R20](09-risks.md) is the highest-frequency
delivery risk and it is a staffing question, not a technical one. Needs names and
a number of hours per week, endorsed by department heads, before Phase 2 exit.

### Q11
**May one person hold both `ddi-maker` and `ddi-checker` roles?**
Owner: Clinical governance.
Recommendation: yes. The database trigger prevents self-approval on any
individual proposal regardless, and in a small unit forbidding the role
combination would halt work whenever one person is on leave. The control that
matters is per-proposal, not per-person.

### Q12
**Can a checker approve a proposal made by a resident they directly supervise?**
Owner: Clinical governance.
Not a technical question, but it determines whether `reviewed_by != proposed_by`
is a real control or a formality. Worth an explicit position in the governance
document rather than leaving it implicit.

## Blocking Phase 3

### Q13
**What counts as a "department" for the top-50 exercise** — administrative
department, clinical unit, or speciality? And how many are there?
Owner: Clinical informatics lead.
Drives the 1,250-row and 500–650-molecule estimates. If AIIMS has 40 units rather
than 25, the row count rises but the distinct-molecule count rises much less;
worth confirming since the *calendar* scales with units (liaison sessions) while
the *effort* scales with distinct molecules.

### Q14
**Should the second sort key for curation sequencing be added** — NTI / QT /
CYP-perpetrator status, alongside department count?
Owner: Clinical pharmacology lead.
See [C12](11-challenges-to-the-brief.md#c12). My recommendation: yes. Department
count finds cross-specialty pairs, which is right, but a single-department
antiarrhythmic or an azole is exactly the kind of drug where a missed interaction
causes harm, and frequency ranking buries it.

## Blocking Phase 6

### Q15
**Who owns the interruptive tier, and by what governance process is it changed?**
Owner: Drug and Therapeutics Committee (or institutional equivalent).
This is the most clinically consequential configuration in the system and it must
not be an engineering decision. It needs a standing owner, a change process, and
a review cadence — ideally the existing DTC, which already has the right
membership.

### Q16
**What is the acceptable interruptive alert rate for this institution?**
Owner: DTC.
The plan uses ≤ 2 per 100 orders from the general CDS literature. The institution
may have a different tolerance; the number must be agreed *before* shadow mode so
that the gate is a pre-registered threshold rather than a post-hoc negotiation.

### Q17
**Is ethics committee approval required for the shadow-mode and pilot data
collection?**
Owner: Clinical informatics lead.
Shadow mode logs prescribing patterns. Even with no patient identifiers, an IEC
view should be obtained early; it is usually straightforward and is painful to
retrofit.

## Blocking Phase 7

### Q18
**Which second site?** A district hospital, CHC or PHC must be named, with its
formulary available, to prove the one-binary/two-overlay claim.
Owner: Project lead.
If no second site is committed, the primary-care deployability claim is untested
and should be dropped from the programme's stated objectives rather than
asserted.

### Q19
**Who maintains this after handover, and with what budgeted time?**
Owner: Institutional leadership.
[R26](09-risks.md). The quarterly re-census is ~0.5 person-week plus ~4 clinical
hours. That is small, but it must belong to someone by name. A knowledge base
with no maintainer degrades silently, and a silently degraded DDI checker is
worse than none, because clinicians trust it.
