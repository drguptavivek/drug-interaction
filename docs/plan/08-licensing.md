# 08 — Licensing Analysis

**This is an engineering analysis, not legal advice.** Every conclusion here is a
position to be confirmed by institutional legal counsel in Phase 0, and the
Phase 0 exit criteria require a signed memo. Licence terms are also read here as
of the retrieval date recorded in `sources.lock`; they change, and the text as
retrieved is stored in `licence.retrieved_text` for exactly that reason.

## 1. Source-by-source position

| Source | Licence (to verify) | May we use it? | May we redistribute derived content? | Practical consequence |
|---|---|---|---|---|
| **DDInter 2.0** | CC BY-NC-SA 4.0 | Yes, non-commercial | **Yes, but only under CC BY-NC-SA 4.0** | Drives the whole artifact licensing question — §2 |
| **SNOMED CT International** | SNOMED Affiliate Licence (free at point of use in India as a Member country, via the NRC) | Yes, with an affiliate licence | **Identifiers: yes, in context. Descriptions/FSNs: restricted to licensees** | §3 |
| **SNOMED CT India edition** | NRCeS national licence terms | Yes, for Indian entities | Restricted; national-licence terms govern | §3 |
| **Common Drug Codes for India (CDCI)** — Terminology Integrated Package | Distributed via **MLDS** under the same affiliate/national terms | Yes, for Indian entities | Restricted | §3.4 |
| **CSNOtk / CSNOServ / CSNOFinder / CSNOLib / CSNOCtrl** (the *software*) | **Apache-2.0**, © 2014 C-DAC | Yes | Yes | §3.5 — but the SNOMED **content** it serves is governed separately |
| **ONCHigh high-priority list** | Published in the peer-reviewed literature; licence status of the *list as data* is unclear | Yes | **Unclear — Phase 0 question** | Treat as restricted until confirmed; a ~15-entry list can be independently re-derived from cited primary literature if needed |
| **CredibleMeds QTDrugs** | CredibleMeds Terms of Use — registration required, personal/institutional use | Yes, per site, by the licensee | **No** | §4 — do not ship |
| **DrugBank (academic/full)** | Academic licence agreement | Only under a signed agreement | **No** | §5 — do not use for rules |
| **DrugBank Open Data (vocabulary)** | CC0 1.0 | Yes | Yes | Names/synonyms/UNII cross-references only |
| **openFDA / FDA label data** | US Government work, public domain | Yes | Yes | Evidence display only, not a rule source |
| **UNII / FDA GSRS** | Public domain | Yes | Yes | Anchor |
| **WHO ATC index** | WHO copyright; use permitted, redistribution of the full index restricted | Yes for internal mapping | **Codes yes, full index no** | Ship codes, not the index |
| **RxNorm** (`SAB=RXNORM`) | UMLS Metathesaurus Licence (free, UTS registration) | Yes | RXCUIs and RxNorm term types: yes, with attribution | §5A — promoted to a structural anchor |
| **RxNorm, proprietary source vocabularies** (MMSL, GS, NDDF, MDDB, …) | Source-specific, restricted | Present in the release | **No** | Filtered out at ingest, not merely unused |

## 2. The CC BY-NC-SA ShareAlike obligation — and why the KB must not be embedded

### 2.1 What ShareAlike requires

CC BY-NC-SA 4.0 requires that **Adapted Material** — anything derived from
DDInter — be distributed under the same licence (or a BY-NC-SA-compatible one),
with attribution, a licence notice, an indication of changes made, and no
additional legal or technical restrictions on downstream recipients.

The curated knowledge base is unambiguously Adapted Material. It is built from
DDInter's pair list, severities, mechanism text and management text.

Therefore: **`kb.ddi` must be distributed under CC BY-NC-SA 4.0.** That is fine
and consistent with the project's non-commercial, government-and-academic
posture.

### 2.2 Why `go:embed` is the problem ([C1](11-challenges-to-the-brief.md#c1))

The brief says "single binary, knowledge base embedded **or** loaded from a
read-only local file." Take the second option, and make it the only option.

If the KB is embedded, the compiled binary *contains* Adapted Material. The
binary as distributed is then a single file subject to a ShareAlike obligation,
and several unpleasant things follow:

| Problem | Detail |
|---|---|
| Licence of the combined artifact | The distributed binary carries CC BY-NC-SA content. At minimum the recipient must receive the licence, attribution and change notice *with the binary*, and must be free to extract and redistribute the KB portion under BY-NC-SA. Shipping a stripped binary with no NOTICE does not satisfy that. |
| "No additional restrictions" | Embedding KB content inside a compiled executable is arguably a technical measure that restricts the recipient's exercise of the licensed rights. This is a real, live argument about CC's anti-TPM clause, and it is not one worth having. |
| The NonCommercial term travels | Any recipient of the binary is bound by NC. A binary that cannot be used commercially is a constraint on the *software*, not just the data — an unnecessary entanglement. |
| Update mechanics | Every KB release requires a recompile, which is operationally worse for a fleet of offline PHC installs that need a 14 MB data update, not a 30 MB binary replacement. |
| Source-licence clarity | The Go source can be openly licensed (Apache-2.0 / MIT) *only* if it is separable from the data. Embedding destroys that separation. |

### 2.3 Recommended distribution model

Two artifacts, two licences, distributed together but severable:

```
ddid                      Apache-2.0 (or MIT).  No DDInter-derived content.
                          Contains: code, the Ed25519 public key, no knowledge.

ddi-kb-<version>.tar.zst  CC BY-NC-SA 4.0.
                          Contains: kb.ddi, kb.ddi.sig, manifest.json,
                                    NOTICE, LICENSE-KB, diff report.
```

This gives: a cleanly open-source service anyone may reuse; a knowledge artifact
that honours ShareAlike and NC; no TPM argument; and independent update cadences.
It costs one command-line flag.

`go:embed` may be retained **only** for a hypothetical "open tier" artifact built
exclusively from public-domain sources (openFDA + UNII + ONCHigh, if ONCHigh
clears). That tier would have no DDInter content and therefore no ShareAlike
obligation. Whether it is worth building is [Q8](10-open-questions.md#q8).

### 2.4 ShareAlike and the curation database

A subtler point. `mapping_projection` — moiety ↔ SCTID ↔ DDInter ID — is derived
in part from DDInter (the DDInter ID column and the drug list). Publishing it is
desirable: it is arguably the most reusable output of the whole project and the
thing other Indian institutions most need.

Position: publish the mapping table under CC BY-NC-SA 4.0, with the SNOMED
columns handled per §3. Publishing it is a ShareAlike-compliant act, not a
problem to avoid.

### 2.5 The NonCommercial term and a government hospital

CC's NonCommercial means "not primarily intended for or directed towards
commercial advantage or monetary compensation." A government tertiary hospital
using the service in patient care, including in paying private wards, is not
*primarily* a commercial activity, and this is the ordinary reading. But
"ordinary reading" is not the same as institutional sign-off, and the answer for
a fee-levying private ward at AIIMS should be written down by counsel once
rather than re-argued per site. **Phase 0 deliverable.**

A private hospital wanting to deploy the same artifact is squarely outside NC.
The deployment guide must say so plainly, because it *will* be asked.

## 3. SNOMED CT and the India national licence

### 3.1 What is restricted

SNOMED CT content — concept descriptions (FSNs, preferred terms, synonyms),
relationships, reference sets — is licensed content. Distribution to
non-affiliates is restricted. India is a SNOMED International Member, so SNOMED
CT is available at no cost to Indian users through NRCeS under the national
licence; that makes affiliate status easy to obtain **within India** and does not
make it optional.

The distinction that matters in practice:

| Artifact element | Position |
|---|---|
| **SCTIDs alone**, used as codes in an interface or a data payload | Permitted in ordinary use; this is what codes are for |
| **SCTID + FSN/description** in a published mapping file | This is SNOMED content. Distribute to affiliates; do not publish openly without checking the national licence terms |
| A **derivative** (our substance sub-hierarchy, the `Is modification of` closure) | A SNOMED derivative; distribution restricted |
| The **runtime artifact** containing SCTIDs *and* FSN snapshots for display | Restricted content inside the artifact — §3.2 |

### 3.2 Consequence for `kb.ddi`

The artifact needs SCTIDs (for resolution) and will want FSN snapshots (so that
`/v1/ingredients/{id}` can display a recognisable name and so that findings read
sensibly).

Two build profiles, selected at build time:

| Profile | Contents | Distributable to |
|---|---|---|
| `full` | SCTIDs + FSN snapshots + salt hierarchy | SNOMED affiliates (i.e. Indian institutions under the national licence) — the normal case |
| `codes-only` | SCTIDs + our own `preferred_name` (curated, not SNOMED-derived) | Anyone, subject only to the DDInter CC BY-NC-SA terms |

`preferred_name` in the `ingredient` hub is deliberately *our* name, curated by
the maker, not a copy of the FSN. That is not bureaucratic hygiene — it is what
makes the `codes-only` profile possible at all, and it should be enforced during
curation (the UI must not offer "copy FSN" as a one-click action).

The artifact builder takes `--profile` and the `verify` stage asserts that a
`codes-only` build contains no SNOMED-derived strings.

### 3.4 Common Drug Codes for India (CDCI) — what it actually is

Confirmed (August 2026 release note, via NRCeS/MLDS):

| | |
|---|---|
| Form | A **Terminology Integrated Package** — RF2, distributed through MLDS |
| Contents | Generic medicines, **supplier** concepts, and **branded** medicine concepts |
| Coverage claim | With the SNOMED CT International Release, covers all medicines **except devices, surgical implants and combi packs** |
| Cadence | Synchronous with the SNOMED CT International Edition (the August 2026 package pairs with the August 2026 International Edition) |
| Recent increment | 134 Clinical drug (CD) and 892 Real clinical drug (RCD) concepts added, driven by national programmes |

This settles [Q4](10-open-questions.md#q4) favourably: CDCI is real SNOMED
drug-model content with `has active ingredient` /
`has precise active ingredient` / `has basis of strength substance` structure,
not a CSV of product names. Product decomposition works as designed, and the FDC
parsing project I had budgeted as a downside risk does not arise for covered
products.

Two caveats that are now design inputs rather than unknowns:

- **Combi packs are excluded.** A combi pack (several distinct dose forms in one
  pack — H. pylori kits, TB kits, peri-operative kits) has no single concept, so
  such a drug master row cannot be coded to one code and must be decomposed into
  its component products by the coding exercise. This is a named check in
  `ddictl qa-coding` ([13 §4.3](13-hmis-neutral-integration.md#43-drug-master-coding-qa-report)).
- **Branded (RCD) coverage is being built out incrementally.** An increment of
  892 RCD concepts, targeted at national programmes, is not the same as covering
  a tertiary hospital's branded formulary. Expect a mix: some drug master rows
  code to RCD, many to substance level.

### 3.5 CSNOServ: Apache-2.0 software, separately licensed content

C-DAC's toolkit **CSNOtk** — `CSNOLib`, `CSNOFinder`, `CSNOServ`, `CSNOCtrl` — is
**Apache-2.0** (© 2014 C-DAC). The SNOMED CT content it serves is governed by a
separate C-DAC-issued SNOMED CT sub-licence, whose clause 4.2 forbids a
non-Affiliate from using it to *"add or copy SNOMED CT identifiers into any type
of record system, database or document"*.

That is precisely what the curation platform does. So **SNOMED International
Affiliate registration (free to Indian organisations, via MLDS) is a prerequisite
for using CSNOServ or BHTS in the curation workflow at all** — not merely for
distributing the result. It gates Phase 2, not Phase 7. Detail and the other
clauses in [12 §A7](12-terminology-tooling.md#a7-the-csnoserv-sub-licence-constrains-how-we-may-use-it).

Clause 5.1 also places responsibility for fees for deployment in a **Non-Member
Territory** on the Affiliate. Deployment within India is free of cost; offering
the artifact to a neighbouring country needs that check first, and the deployment
guide should say so before someone offers it informally.

### 3.3 The India edition dependency

**[Q2](10-open-questions.md#q2) is now answered.** The CDCI package description —
generic, supplier and branded *medicine* concepts, used *alongside* the
International Release to cover all medicines — confirms the expectation:
substance concepts are International Release core content, and the Indian
extension adds the *product* layer. This is the good outcome. The substance spoke
targets are internationally stable concepts, and the India extension dependency
is confined to products, which is exactly where the local variability belongs.

The startup validation must then check against the **combined** release
(International + India extension) and record both release identifiers, not just
the India one as the brief states.

## 4. CredibleMeds — do not ship ([C2](11-challenges-to-the-brief.md#c2))

CredibleMeds makes the QTDrugs lists available to registered users under terms
that, on my reading, do not permit redistribution or incorporation into a
redistributed product. The brief places it in the "always alert" tier of a
shipped artifact. That is the one licensing position in the brief I would call
clearly unsafe.

Design accommodation — and it is a small one:

- The KB defines a **TdP risk flag** on the ingredient record, with an empty
  value in the shipped artifact.
- Each deploying institution obtains its own CredibleMeds registration and
  supplies `tdp-overlay.yaml`, mapping ingredient `public_id` → risk category.
- The service loads it as an overlay, exactly like the formulary overlay, and
  reports `tdp_source: "site-supplied"` in responses.
- `ddictl` ships a converter that turns a CredibleMeds export into the overlay
  format **on the site's own machine**, never in our build.

This costs about 0.3 eng-weeks and removes the exposure entirely. It also has a
genuine clinical benefit: QT risk categorisation changes more often than DDI
data, and a site-refreshable overlay updates without a KB release.

If Phase 0 finds that CredibleMeds *does* permit redistribution for
non-commercial public-health use, the overlay mechanism still works and simply
gets pre-populated. The design is right either way.

## 5. DrugBank ([C7](11-challenges-to-the-brief.md#c7))

Two separate things share the name and must not be confused:

| | DrugBank Open Data | DrugBank academic / full release |
|---|---|---|
| Content | Drug names, synonyms, external identifiers (UNII, CAS) | Full database including interactions |
| Licence | CC0 1.0 | Signed academic licence; redistribution not permitted |
| Usable here | **Yes** — vocabulary for retriever R5 and for name normalization | **No** — cannot ship derived DDI content |

The brief proposes DrugBank for "coverage gap-filling." Two problems:

1. **Legal.** Redistributing DDI content derived from the academic release in a
   shipped artifact is not permitted under that agreement.
2. **Circular.** DDInter is itself substantially DrugBank-derived. Gap-filling
   DDInter with DrugBank therefore adds far less coverage than the raw record
   counts suggest, while adding a licence entanglement. The Phase 0 coverage
   census should measure this directly: take the molecules *missing* from
   DDInter and check how many are present in DrugBank with interaction content.
   My expectation is that the overlap is high and the incremental yield is small.

Recommendation: use DrugBank Open Data (CC0) for vocabulary only. Drop DrugBank
as a rule source. If the census shows a genuinely large gap, the honest options
are an institution-authored rule set (fully ours, freely publishable) or a
licensed commercial source — not a licence-breaching workaround.

## 5A. RxNorm

Two things ship in one download and must be separated at ingest.

| | RxNorm proper (`SAB=RXNORM`) | Proprietary source vocabularies in the same release |
|---|---|---|
| Examples | RXCUIs, `IN`/`PIN`/`MIN`/`SCD` term types, RxNorm normal forms | MMSL, GS, NDDF, MDDB |
| Redistributable | Yes, with NLM attribution | **No** |
| Used here | Structural anchor path, IN/PIN collapse cross-check, UNII derivation | Nothing |

Mechanism: filter to `SAB=RXNORM` at ingest — proprietary atoms never enter the
staging database at all, so they cannot leak into an artifact by oversight. The
`verify` stage asserts it independently. Brand term types (`BN`, `SBD`) are
discarded for a separate reason: they are a false-evidence source for an Indian
formulary ([12 §B4](12-terminology-tooling.md#b4-the-tightening-ingredient-level-only-never-brand)),
not a licensing problem.

Why RxNorm's role grew: it may be the only available derivation of the
SNOMED-side UNII anchor, and it provides an independent opinion on salt/moiety
collapse. See [12 Part B](12-terminology-tooling.md#part-b--rxnorm).

## 3.6 Affiliate licensing in practice: who needs one, and what it costs

Source: SNOMED International's published Affiliate licensing guidance (as
retrieved; store the text in `licence.retrieved_text` per [§1](#1-source-by-source-position)).

### 3.6.1 India is a Member country — the licence is free

Within India, the Affiliate Licence is **free of cost**, obtained from NRCeS.
There is no fee to resolve, no Territory Band to look up, no invoice. The
obligation is **registration and annual reporting**, not money. Fees arise only
for use, deployment or distribution in **non-Member** countries, priced by the
country's World Bank Territory Band; exemptions exist for low-income countries,
development licences, qualifying research projects and humanitarian use.

The rule that matters for architecture: **the licence follows where SNOMED CT is
USED, not where it is hosted.** Hosting location is irrelevant.

### 3.6.2 We are not SaaS, and that is a licensing advantage

SNOMED International's guidance treats SaaS specially: a service offered to
healthcare organisations as SaaS must be declared as such on MLDS, and is
**charged per site** (clinic, hospital) where fees apply.

This plan's architecture — a binary plus a signed data file, deployed and run by
each institution, with no network dependency — is **not** SaaS. Each deploying
institution is the user of SNOMED CT in its own right. Within India that means
each is eligible for its own free Affiliate Licence from NRCeS, and no per-site
fee arises anywhere.

Worth noting because it was chosen for clinical and connectivity reasons
([05](05-go-service.md)) and happens to be the cleaner licensing posture too. If
anyone later proposes "simplify this into a hosted national DDI API", the
licensing consequence — a SaaS declaration and per-site accounting — belongs in
that discussion.

### 3.6.3 The 2023 update: downstream systems do not need a licence

The 2023 Affiliate Licence update clarifies what non-licensed systems may do:

| Actor | Permitted |
|---|---|
| User of a **SNOMED-licensed** system | Transmit SNOMED codes **and descriptions** to any system, licensed or not — no restrictions |
| User of a **non-licensed** system | **Receive** codes and descriptions; **store** them in their system and data repository; **forward** them to licensed and unlicensed systems |

Two consequences, and the first corrects a position taken earlier in this plan:

1. **Our API may return SNOMED descriptions to any consumer.** Earlier drafts were
   cautious about putting FSN text in responses. Transmission of codes *and
   descriptions* from a licensed system is unrestricted, so a finding may name a
   substance using SNOMED text without imposing a licence obligation on the
   caller. This is data flow in use.
2. **It does not make publication unrestricted.** Receiving, storing and
   forwarding codes in the course of use is different from *distributing the
   terminology* — publishing a downloadable mapping file containing FSNs is
   distribution and remains restricted. The `codes-only` build profile
   ([§3.2](#32-consequence-for-kbddi)) is still the mechanism for anything
   published for download.

### 3.6.4 Where the boundary actually falls

Four different things get conflated. They have different answers:

| Activity | Example in this project | Licence needed? |
|---|---|---|
| **Creating** SNOMED-coded data — browsing the terminology and assigning codes | Our curation platform; the HMIS drug master coding exercise ([13 §5](13-hmis-neutral-integration.md#5-the-drug-master-coding-exercise)) | **Yes** — a Data Creation System |
| **Deploying** software that contains SNOMED CT content | A site running `ddid` with a `full`-profile `kb.ddi` (SCTIDs + FSN snapshots) | **Yes** — this is deploying SNOMED CT |
| **Deploying** software that holds only codes | A site running `ddid` with a `codes-only` `kb.ddi` (SCTIDs + *our* curated names) | **Probably not** — arguably just storing codes, per §3.6.3. **[Q29](10-open-questions.md#q29)** |
| **Receiving, storing, forwarding** codes | An HMIS storing SCTIDs returned in a finding; a downstream record system | **No** — explicitly permitted |

The third row is the one with real operational weight. AIIMS will hold an
Affiliate Licence regardless, because it is doing the coding. But a rollout to
hundreds of CHCs and PHCs — none of which code anything, they only run the
binary — is a very different proposition if each needs its own registration
versus none needing one.

**This gives the `codes-only` profile a second, independent justification.** It
was introduced to satisfy redistribution limits ([§3.2](#32-consequence-for-kbddi));
it may also be what makes a large primary-care rollout administratively feasible.
The design already supports both profiles, so the answer to
[Q29](10-open-questions.md#q29) changes which profile primary care ships with, not
the architecture.

### 3.6.5 Annual Declaration of Use — a recurring obligation the plan had missed

Affiliate licence holders must submit or renew a **Declaration of Use annually,
by 15 January**, via MLDS (or via the NRC, for countries with their own
distribution service — so **via NRCeS** for us). It must report:

- current and planned use of SNOMED CT, and the purpose of use;
- implementation status of the software;
- **sublicensees and end users** (organisations or individuals);
- **the number of software applications and workstations**;
- the type of usage (data collection, evaluation, aggregation).

This is a genuine recurring obligation and it has a design consequence. The
service is deliberately offline with no telemetry and no phone-home, so
deployment counts **cannot** be collected technically — and adding telemetry to
collect them would breach the no-network principle for a purely administrative
purpose. It must therefore be an **administrative deployment register**,
maintained by the institutional owner:

| Field | Why |
|---|---|
| Site name, type (tertiary / district / CHC / PHC), state | "End users" and site counts |
| KB version and build profile (`full` / `codes-only`) deployed | Determines whether that site is deploying SNOMED content |
| Number of workstations or service instances | Required reporting field |
| Date deployed, date last updated | Implementation status |
| Contact | Withdrawal notices ([02 §12](02-data-model.md#12-releases)) |

The register does double duty: it is also the distribution list for a withdrawn
release, which [05 §6](05-go-service.md#6-startup-validation) otherwise handles
only by shipping a `withdrawn.txt` with each update — a mechanism that by
definition cannot reach a site that never updates.

**Ownership.** This lands on whoever takes handover ([Q19](10-open-questions.md#q19)).
It is perhaps two hours a year, but a missed declaration is a licence compliance
failure, not a paperwork slip. Added to the Phase 7 handover checklist and to the
risk register as [R30](09-risks.md).

### 3.6.6 Obtaining it — **already done for this project**

**Status: the Affiliate Licence is held and MLDS access is in place.** This
removes what would otherwise have been the first blocker in Phase 0
([Q28](10-open-questions.md#q28) closed). The procedure is recorded below because
it will be needed again — by a second deploying site, and by any other Indian
institution reusing this plan.

Per NRCeS (`nrces.in/standards/snomed-ct`):

| Step | |
|---|---|
| 1 | Register at MLDS, India landing page — `mlds.ihtsdotools.org/#/landing/IN` |
| 2 | Create and activate the account |
| 3 | Accept the **SNOMED CT Affiliate License Agreement (2023)** and request access |
| 4 | **Download link available in 4–5 business days** |

The 4–5 business day wait matters for anyone doing this fresh: it sits at the
head of a serial chain — no licence → no release files → no unpacked RF2 → no
coverage census. For this project that chain is already cleared, so Phase 0 can
open directly on the downloads.

**One check still outstanding: the national releases, not only the International
Edition.**
The NRCeS page describes the International Release files as what becomes
available to the affiliate. The **Common Drug Codes for India (CDCI)** package —
which this project depends on for the entire product layer
([§3.4](#34-common-drug-codes-for-india-cdci--what-it-actually-is)) — is a
separate national release, listed at `nrces.in/services/national-releases`.
Confirm CDCI is available under the existing registration; an International
Edition download does not imply access to the national packages. If it is not,
request it now — it is the only remaining licensing item on the critical path.

While there, capture the Affiliate License Agreement (2023) PDF verbatim into
`licence.retrieved_text` with its retrieval date ([02 §2](02-data-model.md#2-provenance)) —
it is the document every position in this chapter rests on.

**Release cadence.** SNOMED CT International ships **biannually**, and CDCI is
synchronous with it. That is the external clock this project's terminology runs
on, and it is the natural cadence for the re-census (§3.6.5,
[C6](11-challenges-to-the-brief.md#c6)).

### 3.6.7 If this ever leaves India

Not in scope, but the rules are worth recording before someone offers the
artifact informally to a neighbour:

- **Another Member country** → a separate Affiliate Licence from *that* country's
  NRC. Licences do not travel; multi-country use means multiple licences.
- **A non-Member country** → a licence from SNOMED International, with a fee by
  Territory Band, renewed annually. Exemptions may apply (low-income country,
  humanitarian, qualifying research).
- **Genuinely multi-country from the outset** → SNOMED International expects to
  be contacted about a global licence.

## 6. What may be published openly

| Artifact | Licence | Publishable |
|---|---|---|
| `ddid`, `ddictl`, ETL code, Svelte UI | Apache-2.0 | **Yes, openly** |
| API specification (OpenAPI, CDS Hooks discovery) | CC BY 4.0 | **Yes** |
| This plan and all design documentation | CC BY 4.0 | **Yes** |
| `mapping_projection`, `codes-only` profile (our names + SCTIDs + DDInter IDs) | CC BY-NC-SA 4.0 | **Yes** — high reuse value for other Indian institutions |
| SNOMED descriptions **returned in API responses** at runtime | n/a — transmission, not distribution | **Yes** — unrestricted from a licensed system, and the receiving system needs no licence of its own ([§3.6.3](#363-the-2023-update-downstream-systems-do-not-need-a-licence)) |
| `mapping_projection`, `full` profile (with FSNs) | CC BY-NC-SA 4.0 + SNOMED affiliate terms | To affiliates |
| Exception rules and their rationales | CC BY-NC-SA 4.0 | **Yes** — arguably the most valuable curated output |
| Validation set (pairs + expectations, de-identified) | CC BY 4.0 | **Yes**, with departmental consent |
| Alert tier definitions | CC BY-NC-SA 4.0 | **Yes** |
| `kb.ddi` full artifact | CC BY-NC-SA 4.0 + SNOMED terms | To affiliates / Indian institutions |
| Shadow-mode and pilot logs | — | No (institutional data; aggregate statistics only) |
| CredibleMeds-derived content | — | **Never** |

## 7. Compliance mechanics, not compliance promises

Three mechanisms, all in code, so that compliance does not depend on anyone
remembering:

1. **`source_release.redistributable`** — the artifact builder refuses to emit
   any row traceable to a non-redistributable source ([04 §7](04-etl-pipeline.md#7-verify)).
2. **`--profile codes-only` assertion** — verify fails if any SNOMED-derived
   string is present in a codes-only build.
3. **NOTICE generation** — the `NOTICE` file and the `META` section are generated
   from `licence.attribution_text`, so attribution cannot drift from the sources
   actually used. An artifact separated from its tarball still carries its own
   licence text, which is what ShareAlike requires.

## 8. Attribution text (draft for `NOTICE`)

```
This knowledge base is Adapted Material under CC BY-NC-SA 4.0.

Contains information from DDInter 2.0, used under CC BY-NC-SA 4.0.
  Changes made: drugs mapped to SNOMED CT substance concepts and to an
  internal moiety model; salt-form entries collapsed to moiety except where
  flagged; interaction pairs restricted to the mapped subset; severity
  retained unchanged; mechanism and management text retained unchanged.

Contains SNOMED CT content, used under the SNOMED CT Affiliate Licence via
  the National Release Centre for India (NRCeS). SNOMED CT is a registered
  trademark of SNOMED International. This artifact may only be used by
  parties holding a valid SNOMED CT licence.

Contains RxNorm content courtesy of the U.S. National Library of Medicine
  (RXCUIs and RxNorm term types only; no proprietary source vocabulary content).
Contains UNII data and FDA drug label data, which are in the public domain.
Contains ATC codes, © World Health Organization Collaborating Centre for
  Drug Statistics Methodology.

This knowledge base is licensed CC BY-NC-SA 4.0. Commercial use is not
  permitted. Derived works must be shared under the same licence.

This artifact does NOT contain CredibleMeds QTDrugs data. TdP risk
  categorisation, if present at runtime, is supplied by the deploying
  institution under its own CredibleMeds registration.
```
