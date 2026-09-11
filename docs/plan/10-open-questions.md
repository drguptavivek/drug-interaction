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
**Does DDInter 2.0 actually contain drug–food interactions, drug–disease
interactions and therapeutic duplication?**
Blocks: scope, and the Phase 6/7 stakeholder expectations.
Owner: Clinical pharmacology lead + engineering.
See [C3](11-challenges-to-the-brief.md#c3). My expectation is that DDInter is
drug–drug only. If stakeholders have been promised the other three, either a
source must be found or the scope must be reduced explicitly and in writing.
Default if unanswered: descope to drug–drug, plus structural moiety-duplication
detection clearly labelled as structurally derived.

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
consideration. See [12 §A3](12-terminology-tooling.md#a3-which-server--revised-after-the-nrces--c-dac-findings).

### Q27
**Does CSNOServ's API support ECL?**
Blocks: whether Snowstorm is needed at all.
Owner: Terminologist + engineering. **~Half a day against the live endpoint.**
ECL was the whole case for standing up Snowstorm ([12 §A2](12-terminology-tooling.md#a2-build-and-curation-time-yes-and-it-is-better-than-my-first-draft)).
If CSNOServ does ECL — via FHIR `ValueSet/$expand` with an ECL filter, or
natively — the 2.0 eng-weeks budgeted for a Snowstorm standup and the 16 GB build
machine both disappear. **Check the API, not the browser**: CSNOFinder browsing
works regardless and tells you nothing about ECL. Full check-list in
[12 §A5](12-terminology-tooling.md#a5-what-to-check-before-committing-phase-0-half-a-day).

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
**Does the SNOMED release we will use ship a UNII map reference set?**
Blocks: the anchor design's SNOMED-side derivation path.
Owner: Terminologist.
Why it matters more than it looks: the third-anchor check is the control that
detects spoke disagreement, and it needs UNII computed *independently* down the
SNOMED side. If no UNII refset exists, that derivation has to come from RxNorm
(`SCTID → RXCUI → UNII`), which makes RxNorm structurally necessary rather than
optional. See [12 §B1](12-terminology-tooling.md#b1-why-rxnorm-may-be-load-bearing-not-optional).
Default if unanswered: ingest RxNorm regardless and treat it as the primary
derivation.

### Q23
**Does DDInter publish DrugBank cross-references, and does the RxNorm release
carry `DRUGBANK` as a source vocabulary?**
Blocks: retriever R8, the structural `DDInter → DrugBank → RXCUI → SCTID` path.
Owner: Engineering.
If both hold, spoke B links to spoke A with no string matching at all — much
stronger evidence than any name-similarity score. If either fails, R8 degrades
to name matching and nothing else in the design changes.

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
**Does DDInter's bulk download cover all 14 ATC first levels, or only 8?**
Blocks: **the whole project.** This is the coverage question in its sharpest
form.
Owner: Engineering + clinical pharmacology. **Half a day.**

The download page at `ddinter.scbdd.com` lists 8 files — A, B, D, H, L, P, R, V
— missing **C, G, J, M, N, S**. Those are cardiovascular, anti-infectives,
nervous system, musculoskeletal, genito-urinary and sensory organs: the classes
containing most CYP perpetrators, most narrow-therapeutic-index drugs, and most
QT prolongers.

**Check `ddinter2.scbdd.com` first** — that is DDInter 2.0; `ddinter.scbdd.com`
is 1.0, and the observed file set may simply be the older release's.

If the gap is real on 2.0 as well: pairs survive when *either* partner is in a
present class, so what is lost is pairs with **both** partners in missing classes
— azole × statin, SSRI × tramadol, phenytoin × carbamazepine, amiodarone ×
haloperidol. Fewer pairs than 6-of-14 implies, but concentrated in the severe
ones, which makes the [P7](14-phase-0-runbook.md#p7--coverage-census--the-gate)
NTI/QT stop condition the binding constraint.

Default if the gap is confirmed: this is a **re-scope trigger**, not something to
engineer around. The honest options are a curated high-priority-list service
(ONCHigh + institution-authored rules), an institution-authored rule set for the
missing classes, or a licensed commercial source.

### Q32
**Does the DDInter bulk CSV carry mechanism and management text, or only drug
pairs and a severity level?**
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
