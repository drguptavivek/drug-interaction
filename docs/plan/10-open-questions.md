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
**Does the SNOMED CT India edition contain substance concepts, or only products
and dose forms layered on the International Release?**
Blocks: Phase 1 loader design, startup validation scope.
Owner: Terminologist.
Consequence: if substances are International core, spoke A targets are
internationally stable concepts and the India dependency is confined to the
product layer — the better outcome. The startup check must then validate against
*both* release identifiers, not just the India edition as the brief states.

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
**Is CDC-India obtainable in bulk, with structured composition (ingredient,
salt, strength, basis of strength), or only as an online lookup?**
Blocks: Phase 1 product ingestion, Phase 3 FDC tooling estimate.
Owner: Engineering + NRCeS contact.
Default if unanswered: seed `product` from the AIIMS formulary master and attach
CDC-India codes opportunistically. The hub-and-spoke design isolates this, but
the FDC decomposition estimate triples if composition must be parsed from names.

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
**Does NRCeS already hold an Ontoserver licence, or any other FHIR terminology
server, that we could use for curation?**
Blocks: Phase 1 tooling choice.
Owner: Terminologist + NRCeS contact.
Recommendation: Snowstorm (Apache-2.0, free, what SNOMED International runs)
unless an Ontoserver licence already exists. **Do not procure one for this
project** — the benefit over Snowstorm does not justify a purchase here.
See [12 §A3](12-terminology-tooling.md#a3-which-server).

### Q21
**Does NRCeS expose a hosted Snowstorm or FHIR terminology endpoint to Indian
affiliates?**
Blocks: nothing, but it could remove the ops cost of a self-hosted server for
interactive curation.
Owner: Terminologist.
If yes: use it for curation. Still pin RF2 archives for the build — a hosted
endpoint can change release under you mid-project, which is fine for a human
browsing and fatal for a reproducible build.

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
