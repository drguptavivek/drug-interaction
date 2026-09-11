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
| **SNOMED CT India edition / CDC-India** | NRCeS national licence terms | Yes, for Indian entities | Restricted; national-licence terms govern | §3 |
| **ONCHigh high-priority list** | Published in the peer-reviewed literature; licence status of the *list as data* is unclear | Yes | **Unclear — Phase 0 question** | Treat as restricted until confirmed; a ~15-entry list can be independently re-derived from cited primary literature if needed |
| **CredibleMeds QTDrugs** | CredibleMeds Terms of Use — registration required, personal/institutional use | Yes, per site, by the licensee | **No** | §4 — do not ship |
| **DrugBank (academic/full)** | Academic licence agreement | Only under a signed agreement | **No** | §5 — do not use for rules |
| **DrugBank Open Data (vocabulary)** | CC0 1.0 | Yes | Yes | Names/synonyms/UNII cross-references only |
| **openFDA / FDA label data** | US Government work, public domain | Yes | Yes | Evidence display only, not a rule source |
| **UNII / FDA GSRS** | Public domain | Yes | Yes | Anchor |
| **WHO ATC index** | WHO copyright; use permitted, redistribution of the full index restricted | Yes for internal mapping | **Codes yes, full index no** | Ship codes, not the index |
| **RxNorm** | UMLS licence (free, registration) | Yes | Identifiers yes, with attribution | Secondary only |

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

### 3.3 The India edition dependency

Open question for Phase 0 ([Q2](10-open-questions.md#q2)): the brief assumes the
India edition is where the substance concepts live. My expectation is that
substance concepts are International Release core content and the India extension
adds Indian *products* and dose forms on top. If that is right, it is good news —
the substance spoke targets are internationally stable concepts, and the India
extension dependency is confined to the product layer, which is exactly where we
want the local variability to be.

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

## 6. What may be published openly

| Artifact | Licence | Publishable |
|---|---|---|
| `ddid`, `ddictl`, ETL code, Svelte UI | Apache-2.0 | **Yes, openly** |
| API specification (OpenAPI, CDS Hooks discovery) | CC BY 4.0 | **Yes** |
| This plan and all design documentation | CC BY 4.0 | **Yes** |
| `mapping_projection`, `codes-only` profile (our names + SCTIDs + DDInter IDs) | CC BY-NC-SA 4.0 | **Yes** — high reuse value for other Indian institutions |
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

Contains UNII data and FDA drug label data, which are in the public domain.
Contains ATC codes, © World Health Organization Collaborating Centre for
  Drug Statistics Methodology.

This knowledge base is licensed CC BY-NC-SA 4.0. Commercial use is not
  permitted. Derived works must be shared under the same licence.

This artifact does NOT contain CredibleMeds QTDrugs data. TdP risk
  categorisation, if present at runtime, is supplied by the deploying
  institution under its own CredibleMeds registration.
```
