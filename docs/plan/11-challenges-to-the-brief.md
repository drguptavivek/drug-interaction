# 11 — Challenges to the Brief

The brief asked me to flag constraints I believe are mistakes rather than plan
around them. Twelve, ordered by severity. Most of the brief is sound — the
hub-and-spoke model, append-only curation with maker-checker, moiety-level
attachment, the alert-fatigue posture, the outcome taxonomy, and the offline
single-binary deployment are all correct and I have not second-guessed them.

---

## C1
### "Single binary, knowledge base embedded" is incompatible with CC BY-NC-SA

**In the brief:** *"Service in Go. Single binary, knowledge base embedded or
loaded from a read-only local file."*

**Problem.** The knowledge base is Adapted Material under CC BY-NC-SA 4.0. If it
is embedded via `go:embed`, the distributed binary contains ShareAlike-obligated
content. Consequences:

- The binary must ship with the licence, attribution and change notice, and
  recipients must be free to extract and redistribute the KB portion under
  BY-NC-SA. A bare binary does not satisfy that.
- CC's prohibition on "effective technological measures" makes burying licensed
  content inside a compiled executable an argument you do not want to have.
- The NonCommercial term then travels with the *software*, not just the data,
  which needlessly restricts the Go code.
- Operationally it is worse: every knowledge update becomes a recompile and a
  30 MB binary replacement instead of a 14 MB data file — a real cost for a fleet
  of offline PHC installs updated by USB stick.

**Recommendation.** Take the second option in the brief and make it the only one:
`ddid` under Apache-2.0 with no knowledge content; `ddi-kb-<version>.tar.zst`
under CC BY-NC-SA 4.0. One command-line flag. Full reasoning in
[08 §2](08-licensing.md#2-the-cc-by-nc-sa-sharealike-obligation--and-why-the-kb-must-not-be-embedded).

**Severity: blocking (legal).** Decide in Phase 0, before any packaging work.

---

## C2
### CredibleMeds data cannot be shipped in the artifact

**In the brief:** *"ONCHigh high-priority DDI list plus CredibleMeds TdP risk —
the 'always alert' tier."*

**Problem.** CredibleMeds makes the QTDrugs lists available to registered users
under terms that, on my reading, do not permit redistribution or incorporation
into a redistributed artifact. The brief places it inside a shipped knowledge
base. This is the one licensing position in the brief I would call clearly
unsafe — the others are arguable; this one probably is not.

**Recommendation.** Ship an empty TdP risk field. Each site registers with
CredibleMeds itself and supplies `tdp-overlay.yaml`; `ddictl` provides a
converter that runs on the site's machine. ~0.3 eng-weeks, and it has an
independent benefit: QT categorisation changes more often than DDI data, so a
site-refreshable overlay updates without a KB release.

Verify in Phase 0 ([Q5](10-open-questions.md#q5)) — but build the overlay
mechanism regardless, because it is the better design even if redistribution
turns out to be permitted.

**Severity: blocking (legal).**

---

## C3
### DDInter probably does not contain drug–food, drug–disease or duplication data

**In the brief:** *"DDInter 2.0 … ~302,000 DDI records over 2,310 drugs, with
severity, mechanism descriptions and management text; also drug–food
interactions, drug–disease interactions and therapeutic duplication."*

**Problem.** My understanding is that DDInter is a drug–**drug** interaction
resource: drug pairs with severity, mechanism and management. Drug–food,
drug–disease and therapeutic duplication are the content categories that
distinguish *commercial* interaction databases. I cannot verify this from here
and I may be wrong, but the claim should not be load-bearing without checking.

The record counts should likewise be treated as hypotheses to assert at ingest,
not facts — the ETL records actual counts in `source_release.row_counts` and
compares them on every rebuild.

**Why it matters.** If stakeholders have been told the service will cover four
content categories and the source supplies one, that discovers itself in Phase 6
in front of clinicians, which is the worst possible moment. It also changes the
effort estimate materially: sourcing drug–disease content is a separate project
(+6–10 eng-weeks, +4 clinical-weeks, *and a source*).

**Recommendation.** Phase 0 verification ([Q3](10-open-questions.md#q3)). If
confirmed, descope explicitly and in writing. Note that exact-moiety therapeutic
duplication can still be detected *structurally* at near-zero cost — worth doing,
provided it is labelled as structurally derived rather than presented as sourced
knowledge.

**Severity: blocking (scope).**

---

## C4
### Salt→moiety collapse must not be fully automated from `Is modification of`

**In the brief:** *"Salt to moiety collapse is derived from the `Is modification
of` relationship, not hand-curated."*

**Problem.** `738774007 | Is modification of |` is broader than "is a salt of".
It also links **esters, prodrugs and complexes** to their parent substances.
Collapsing all of them to the parent is wrong in exactly the cases where it is
dangerous:

| Pair | `Is modification of` holds | Collapse correct? |
|---|---|---|
| Pantoprazole sodium → pantoprazole | yes | Yes — a true salt |
| Methylprednisolone acetate → methylprednisolone | yes | Usually, but the depot kinetics differ |
| Fosphenytoin → phenytoin | yes | Debatable; conversion-dependent |
| Iron sucrose → iron | yes | **No** — a complex; IV, no luminal chelation |
| Testosterone undecanoate → testosterone | yes | Kinetically very different |

The relationship is also incomplete: not every salt concept carries the edge.
So automation produces both false collapses and missed ones.

**There is also an internal inconsistency.** The same brief insists that every
mapping decision goes through maker-checker because these judgements are
clinical. Salt collapse *is* a clinical judgement — arguably more consequential
than the SCTID choice, since it decides which interaction set attaches. Exempting
it from the workflow that protects everything else is inconsistent.

**Recommendation.** Keep the derivation — as a **proposal generator**. The ETL
classifies each edge into `salt` / `ester` / `prodrug` / `complex` / `unknown`
([04 §3.1](04-etl-pipeline.md#31-substance-graph-and-closure)), proposes
`collapse` for `salt` and `no_collapse` for the rest, and routes everything
through maker-checker. The machine still proposes 100% of rows, so no hand-curation
from scratch; a human still approves, so the clinical judgement is made by a
clinician. That honours the brief's intent — no manual searching — without the
safety exposure.

**Severity: high (clinical safety).**

---

## C5
### Startup validation should fail the release, not the process

**In the brief:** *"a startup validation that every mapped SCTID still resolves as
active in the loaded India extension release, failing loudly on mismatch."*

**Problem.** The check is right. Making it fatal at runtime is not.

The realistic scenario: an administrator upgrades the SNOMED release on a server
without thinking about the DDI service. Three concepts have been inactivated.
`ddid` refuses to start. The HMIS sees a dead endpoint, and — because no EMR
blocks prescribing on a failed DDI callout — clinicians carry on prescribing with
**no** interaction checking and no indication that anything changed. A hard fail
converts a small, visible, partial degradation into a total, silent one.

**Recommendation.** Three tiers
([05 §6](05-go-service.md#6-startup-validation)):

| Failure | `ddictl verify` (CI) | `ddid` runtime |
|---|---|---|
| Signature / CRC / version incompatibility | fail | **fatal** |
| Terminology drift (inactive or missing SCTID) | **fail the release** | `ERROR` log + `degraded` health + per-code `stale_code` + stamped on every response |

"Failing loudly" is preserved — arguably amplified, since a degraded service
announces the problem in `/healthz`, in the log, and in every single API response,
whereas a dead process announces it only to whoever checks systemd. A
`--strict-terminology` flag exists for sites that disagree, defaulting to off and
recorded in the audit log.

**Severity: high (availability, with a safety consequence).**

---

## C6
### "No continuous monitoring" is right for SNOMED and wrong for the formulary

**In the brief:** *"The bulk mapping is a one-time exercise… No continuous
monitoring process."*

**Agreed on the premise.** SNOMED substance concepts are editorially stable,
DDInter major versions are infrequent, and building a monitoring service for them
would be over-engineering. That judgement is correct and I would not change it.

**The gap is elsewhere.** The *Indian formulary* is not stable. CDSCO approvals,
new FDCs, and departmental formulary changes arrive continuously. A KB frozen at
650 molecules quietly stops covering what is actually prescribed, and the failure
mode is `not_evaluated` pairs that nobody is counting.

**Recommendation.** Not a monitoring *service* — two cheap mechanisms already in
the design:

1. `ddi_unresolved_total` and `ddi_pairs_not_evaluated_total` in production. A
   rising unresolved rate *is* the formulary drifting, measured directly, at zero
   marginal cost.
2. A **re-census on a real external clock**. SNOMED CT International ships
   **biannually** and CDCI is synchronous with it, so tie the terminology
   re-census to the release rather than to an arbitrary quarter: re-run the
   coverage census against the current formulary, and pick up whatever new CD/RCD
   concepts the CDCI release adds — each release makes previously uncodeable
   branded rows codeable, which is the drug master's maintenance path too
   ([13 §5](13-hmis-neutral-integration.md#5-the-drug-master-coding-exercise)).
   ~0.5 person-week plus ~4 clinical hours per release. Keep a lighter
   **quarterly formulary check** in between, since CDSCO approvals and
   institutional formulary changes do not wait for SNOMED. Plus the
   source-checksum change gate ([04 §2](04-etl-pipeline.md#2-acquire--source-pinning)),
   which forces a human to look whenever an upstream source moves.

**A third thing the brief missed, and so did my first draft.** There *is* a
mandatory recurring obligation, just not a technical one: the **annual
Declaration of Use**, due 15 January via NRCeS, reporting end users, deployment
counts and implementation status. It needs an administrative deployment register
maintained from the first install — and explicitly *not* telemetry, which would
breach the no-network principle to serve a paperwork need. See
[08 §3.6.5](08-licensing.md#365-annual-declaration-of-use--a-recurring-obligation-the-plan-had-missed).

So the brief's posture survives on the engineering question and needs three
things named: the formulary side, the annual declaration, and an owner for both
([Q19](10-open-questions.md#q19), [Q30](10-open-questions.md#q30)).

**Severity: medium.**

---

## C7
### DrugBank gap-filling is both restricted and largely circular

**In the brief:** *"DrugBank non-commercial dataset — coverage gap-filling only;
it has no severity grading."*

**Two problems.**

1. **Legal.** The DrugBank academic/full release is governed by a signed
   agreement that does not permit redistributing derived content in a shipped
   artifact. "Non-commercial" is not the same as "redistributable". The separate
   *DrugBank Open Data* vocabulary subset is CC0 and is fine — but it contains
   names and identifiers, not interactions.
2. **Value.** DDInter is itself substantially DrugBank-derived. Gap-filling
   DDInter with DrugBank therefore yields far less new coverage than the raw
   record counts imply, while adding a licence entanglement. The brief already
   notes DrugBank has no severity grading, which means anything it adds cannot be
   tiered and lands in the passive bucket — low clinical value for real legal
   exposure.

**Recommendation.** Use DrugBank Open Data (CC0) for vocabulary only — retriever
R5 and name normalization. Drop DrugBank as a rule source. Measure the real gap
in the Phase 0 census: take the molecules missing from DDInter and check how many
DrugBank actually adds. If the gap is genuinely large, the honest answers are an
institution-authored rule set (ours, freely publishable) or a licensed commercial
source — not a licence-breaching workaround.

**Severity: medium (legal + low value).**

---

## C8
### Keycloak at query time contradicts "no network dependency"

**In the brief:** *"No network dependency at query time"* and *"Auth via Keycloak
(existing institutional deployment)."*

**Problem.** These are compatible only with care. Token introspection is a network
call on the request path. At a PHC with no reliable connectivity, or at a district
hospital when the link to the institutional Keycloak is down, a naive integration
denies clinical requests because an *auth* service is unreachable. Denying a DDI
check for an authentication outage is the wrong trade in a clinical setting.

**Recommendation** ([05 §8](05-go-service.md#8-auth-c8)):

- Offline JWT validation against a disk-cached JWKS. No introspection on the
  request path. A stale JWKS warns and continues — the expiry of an auth cache
  must never deny care.
- `mode: static` — a per-site credential file for disconnected sites, rotated by
  the same USB stick that delivers KB updates. Weaker, and honest about it.
- `mode: none` permitted only when the listener is a loopback address. A PHC
  running `ddid` as a sidecar to a single HMIS process on the same box is the
  realistic deployment, and token plumbing there buys nothing.

Keycloak remains the mechanism at AIIMS, as specified. It just cannot be on the
hot path.

**Severity: medium.**

---

## C9
### CDS Hooks is probably not the AIIMS integration path

**In the brief:** *"CDS Hooks `order-select` / `order-sign` service for HMIS
integration, following the HL7 PDDI-CDS implementation guide."*

**Problem.** CDS Hooks assumes the calling system is a FHIR client that can
originate hook calls with a `MedicationRequest` prefetch. Much of the Indian
public-sector HMIS estate — including a large part of the eHospital/NIC
deployment base — is not a FHIR server. ABDM is driving FHIR adoption at the
*exchange* layer (health records, consent), which is not the same as an HMIS
having an internal CDS Hooks client.

If CDS Hooks is the primary surface, there is a real chance of building a
standards-conformant service that the target hospital cannot call.

**Recommendation.** Not "drop CDS Hooks" — build it, to the IG. It has genuine
value for ABDM alignment, for publication, and for sites that can use it. But
sequence it second:

1. REST first — what the AIIMS HMIS will actually be able to call.
2. CDS Hooks second, for conformance and for capable deployments.
3. A thin `ddi-hmis-shim` adapting whatever the HMIS speaks. Budget it, and
   confirm the real protocol in Phase 3, not Phase 5 ([Q1](10-open-questions.md#q1)).

**Update after the HMIS-neutral decision.** This position is strengthened rather
than changed. With SCTIDs going into the HMIS drug master and no site-specific
adapter in our scope, REST is not merely first — it is the only integration
surface we own, and CDS Hooks becomes the standards-conformant option for
deployments that can use it. The `ddi-hmis-shim` I budgeted here is removed from
scope entirely; a conformance kit replaces it. See
[13](13-hmis-neutral-integration.md).

**Severity: medium (sequencing, with schedule risk) — now largely retired.**

---

## C10
### ATC-5 is multi-valued and cannot be an equality anchor

**In the brief:** *"UNII and ATC level 5 are independent verification anchors…
A third anchor (UNII or ATC-5) detects disagreement between the spokes."*

**Problem.** A moiety routinely carries several ATC-5 codes — different codes for
systemic, topical, ophthalmic and combination forms, and sometimes for different
indications. Comparing ATC-5 by equality produces systematic false disagreements
on a large minority of drugs, and an anchor that cries wolf gets ignored, which
costs you the cases where it is right.

There is a related trap on UNII: salts and moieties have **different** UNIIs, and
DDInter identifies drugs sometimes at salt level and sometimes at moiety level. A
naive UNII comparison produces a false disagreement on every salt-listed drug.

**Recommendation.**

- ATC-5: compare by **set overlap** (non-empty intersection), and treat
  disagreement as advisory, not blocking.
- UNII: normalise both sides to moiety via the GSRS parent relationship *before*
  comparing; record the derivation path used in `anchor_observation.derivation`.
  Disagreement after normalisation is genuinely blocking.

Reflected in the schema ([02 §5](02-data-model.md#5-anchors)) and in the ranker
([03 §5](03-candidate-ranking.md#5-anchor-agreement-semantics)).

**Severity: low (correctness), but it degrades a control the design depends on.**

---

## C11
### Route is the general gate, not an iron-specific exception

**In the brief:** *"Known exceptions to moiety collapse: iron and magnesium
preparations, antacid polyvalent cations, oral vs parenteral iron, and
salt-specific DDInter entries."*

**Problem.** Oral-vs-parenteral iron is listed as one exception among four. It is
an instance of a general rule: **systemic exposure depends on route, and
interaction applicability depends on systemic exposure.** Other instances,
unlisted:

| Case | Consequence of moiety-only alerting |
|---|---|
| Topical ketoconazole / clotrimazole | False CYP3A4 alerts — a classic fatigue driver |
| Ophthalmic timolol | The *opposite*: systemic beta-blockade is real; must still alert |
| Inhaled vs oral corticosteroid | Different magnitude |
| Oral (non-absorbed) vancomycin | Systemic interactions do not apply |
| Topical NSAIDs | Far smaller systemic risk than oral |

The ophthalmic timolol row is the important one: this is not "suppress alerts for
non-systemic routes", it is "route determines applicability, in both directions".
A blanket local-route suppression would be its own safety problem.

**Recommendation.** `route` as a first-class attribute on `product`, an
`applicability` predicate on every rule defaulting to systemic × systemic, and a
seeded `EXC_LOCAL_ROUTE` exception class with per-case clinical review. One
schema column and one predicate evaluation. It removes a large class of
alert-fatigue complaints and it generalises the iron exception the brief already
identified.

**Severity: medium.**

---

## C12
### "Top 50 salts per department" under-selects the drugs that matter for DDI

**In the brief:** *"Top 50 drug salts per clinical department… Sequence curation
by number of departments a molecule appears in, highest first."*

**The sequencing rule is right** and I would keep it: cross-specialty molecules
are where interactions are missed, because no single prescriber sees the whole
list. Good instinct, correctly implemented.

**The selection rule has a gap.** Frequency ranking selects high-volume drugs.
DDI risk concentrates in a partly *disjoint* set: narrow therapeutic index
(warfarin, digoxin, lithium, phenytoin, tacrolimus, methotrexate), strong CYP/P-gp
perpetrators (azoles, macrolides, rifampicin, ritonavir), and QT prolongers. Some
of these are high-volume; several are department-specific and moderate-volume,
and frequency ranking pushes them to the tail — where a truncated Phase 3 leaves
them unmapped. A drug prescribed 40 times a month with a major interaction
profile matters more here than one prescribed 4,000 times with none.

**Recommendation.** Keep the top-50 list. Add a second, small, explicitly
risk-ranked list — ask each department for *"any drug you prescribe where you
already check interactions before signing"*, capped at 10 per department. Two
consequences:

1. It surfaces the NTI/QT/perpetrator molecules regardless of volume.
2. It is the same question that generates the validation set, so it costs almost
   no additional departmental time.

Then sequence on `(department_count DESC, risk_flag DESC)`.

**Severity: medium (clinical value).**

---

## What I am not challenging

For the avoidance of doubt, these constraints are good and I have built on them
without modification:

- Hub-and-spoke with an internally minted hub, two independently verified spokes,
  and a third anchor. This is the right shape and it is better than the chained
  mappings most projects of this kind end up with.
- Interactions attach to moiety; salt is a product attribute.
- Append-only proposals and reviews, with the current mapping as a rebuilt
  projection that is never hand-edited.
- `reviewed_by != proposed_by` enforced server-side. I have pushed it further —
  into a database trigger — but the principle is the brief's.
- Batched, numbered releases with a diff report; a single approval mutating
  nothing.
- Pre-computed ranked candidates with evidence, so the human adjudicates rather
  than searches.
- Alert fatigue named as *the* primary failure mode, with a small interruptive
  set and everything else passive.
- Per-institution overlays over one binary.
- `no_interaction_data` distinct from `no_interactions_found`; unresolved drugs
  returned separately and never dropped.
- Go, single binary, no network at query time, read-only knowledge.
- Sequencing curation by department count.

The brief gets the hard architectural judgements right. The dozen items above are
mostly about licences, about one over-automated step, and about a runtime failure
posture — not about the shape of the system.
