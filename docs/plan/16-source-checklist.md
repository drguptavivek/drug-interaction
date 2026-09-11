# 16 — Source Acquisition Checklist

Everything the build needs, what it costs, and what it unblocks. Expansion of
[P5/P6](14-phase-0-runbook.md#p6--acquire-the-remaining-sources).

Every source lands in its own gitignored directory at the repo root
(`snomed-releases/`, `ddinter/`, `rxnorm/`, `umls/`, `drugbank/`,
`credible-meds/`, `cdc-india/`) and is recorded in `sources.lock` with version,
SHA-256, licence and a `redistributable` flag. **`sources.lock` is committed;
the payloads never are.**

## Status

| # | Source | Needed for | Account? | Status |
|---|---|---|---|---|
| 1 | **SNOMED CT International RF2** | Substance hierarchy, spoke A, `Is modification of`, historical associations | MLDS | ✅ **have** |
| 2 | **CDCI / SNOMED CT India** | The entire product layer, FDC decomposition | MLDS | ⚠️ **confirm access** |
| 3 | **DDInter 2.0** | All four content types | none | ⏳ downloading |
| 4 | **RxNorm** | Structural anchor, IN/PIN collapse check, possibly the only UNII path | **UTS — start now** | ❌ |
| 5 | **UNII / FDA GSRS** | Anchor, salt→parent relationship | none | ❌ |
| 6 | **ATC** | Anchor, tiering | *see §2 — do not buy* | ❌ |
| 7 | **NLEM 2022 (India)** | Indian-market prior (F7); PHC/CHC overlay | none | ❌ |
| 8 | **ONCHigh** | The "always alert" tier | none | ❌ |
| 9 | **DrugBank Open Data** | Name/synonym vocabulary only | none | ❌ |
| 10 | **openFDA labels** | Curator evidence display only | none | 🔽 **low priority** |
| 11 | **CredibleMeds** | TdP overlay — **never shipped** | registration | optional |
| 12 | **AIIMS drug master** | [Q24](10-open-questions.md#q24); the coding exercise | institutional | ❌ **ask now** |
| 13 | **CSNOtk / CSNOServ** | Local terminology service (Apache-2.0) | none | after [Q27](10-open-questions.md#q27) |

---

## 1. Start these today — they have lead times

### RxNorm — needs a UMLS Terminology Services account

The account is free but registration is not instant, and RxNorm may be
**structurally necessary**: if the SNOMED release ships no UNII map refset
([Q22](10-open-questions.md#q22)), `SCTID → RXCUI → UNII` is the only derivation
of the SNOMED-side UNII anchor ([12 §B1](12-terminology-tooling.md#b1-why-rxnorm-may-be-load-bearing-not-optional)).

- [ ] UTS account requested at `uts.nlm.nih.gov`
- [ ] **RxNorm Full Monthly Release** (not the weekly, not "current prescribable")
- [ ] At ingest, **filter to `SAB=RXNORM`** — proprietary source vocabularies
      (MMSL, GS, NDDF, MDDB) travel in the same download and must never reach the
      artifact ([08 §5A](08-licensing.md#5a-rxnorm))
- [ ] Discard `BN` / `SBD` brand term types at ingest — Indian and US brand names
      collide for different molecules, so they are a false-evidence source
- [ ] Check `DRUGBANK` is present as a source vocabulary ([Q23](10-open-questions.md#q23))

### CDCI — confirm it is available under the existing MLDS registration

The only remaining licensing item on the critical path. An International Edition
download does not imply access to the national packages.

- [ ] CDCI (Terminology Integrated Package) visible on MLDS, or requested
      (allow 4–5 business days)
- [ ] Version noted, with the International Edition it pairs with — the August
      2026 package is synchronous with the August 2026 International Edition

### AIIMS drug master — the highest information-per-minute item in Phase 0

Not a download, but the thing that collapses most remaining uncertainty
([Q24](10-open-questions.md#q24)). Row count, FDC share, combi-pack share, what
coding it already carries, and whether it records route and dose form.

---

## 2. ATC — the licensing trap, and why you probably do not need to buy it

**Do not purchase the WHOCC ATC/DDD Index.** The online index is free to browse;
the machine-readable file is a paid product, and buying it would be the only
money this project spends on data.

It is very likely unnecessary, because ATC arrives inside sources you already
have:

| Route | Coverage |
|---|---|
| **DDInter** — assigns ATC codes to its drugs, and updated them in 2.0 | Every drug in the interaction set — which is exactly the set needing the anchor |
| **RxNorm** — carries ATC as a source vocabulary | Cross-check, and broader |
| **SNOMED** — an ATC map refset, if the release ships one | Check alongside [Q22](10-open-questions.md#q22) |

- [ ] Confirm ATC codes are present in the DDInter download
- [ ] Confirm ATC is present in RxNorm under the existing UMLS licence
- [ ] Only if both fail, reconsider — and note that ATC is an *advisory* anchor
      compared with UNII ([11 C10](11-challenges-to-the-brief.md#c10)), so a gap
      here is not fatal

Remember ATC is **multi-valued** per moiety. Whichever route supplies it, retain
**all** codes per substance and compare by set overlap, never equality.

---

## 3. Free, no account, fetch when convenient

### UNII / FDA GSRS
Public domain. The UNII list plus the substance relationships — specifically the
**salt → parent moiety** relationship, which is what lets UNII be compared at a
consistent level on both spokes ([03 §5](03-candidate-ranking.md#5-anchor-agreement-semantics)).
Without it, every salt-listed DDInter drug produces a false anchor disagreement.

- [ ] UNII list downloaded
- [ ] Substance relationships included, not just the flat code list

### NLEM 2022 — India's National List of Essential Medicines
Free from MoHFW/CDSCO, usually a PDF, so budget for extraction. Two uses:

- Feature **F7** in the candidate ranker, an Indian-market prior
  ([03 §4](03-candidate-ranking.md#4-stage-3--scoring))
- The seed for the **PHC/CHC formulary overlay** in Phase 6 — without it, the
  primary-care deployability claim has nothing to build on
  ([01 Phase 6](01-phases.md#phase-6--alert-tiering-overlays-clinical-safety))

- [ ] NLEM 2022 obtained and extracted to a table
- [ ] State EDL for the pilot's second site ([Q18](10-open-questions.md#q18)) — Phase 6, not now

### DrugBank **Open Data** only
CC0 vocabulary subset: names, synonyms, external identifiers. **Not** the
academic/full release — that needs a signed agreement and does not permit
redistributing derived interaction content, and it is largely circular with
DDInter anyway ([11 C7](11-challenges-to-the-brief.md#c7)).

- [ ] Open Data vocabulary CSV only
- [ ] Recorded as `DRUGBANK_OPEN`, licence CC0, `redistributable: true`

### ONCHigh
Not really a download — the high-priority list originates in the peer-reviewed
literature. Extract the ~15 class pairs, then **expand class → member** using the
SNOMED substance hierarchy and review the expansion by hand. Class expansion is
how a 15-entry "always alert" list quietly becomes 400 alerts.

- [ ] Source paper obtained; list extracted
- [ ] Licence status of the list *as data* checked ([Q6](10-open-questions.md#q6));
      if unclear, re-derive from the cited primary literature as our own curated
      artifact through maker-checker

---

## 4. Deprioritise

### openFDA — large, and scoped to evidence display only
Several GB of label JSON for something the plan already restricts to
*corroborating evidence shown to curators*, never a rule source — free-text label
mining produces rules nobody can defend at a mortality review
([04 §3](04-etl-pipeline.md#3-normalize)).

Fetch it in Phase 1 if curators ask for it, not in Phase 0. If you do, take only
`drug_interactions`, `contraindications` and `warnings`.

### CredibleMeds — optional, and never shipped
Register only if you want the TdP overlay at AIIMS. It is **site-supplied under
your own registration**, converted locally by `ddictl`, and never enters the
build ([11 C2](11-challenges-to-the-brief.md#c2)). Registering is not a blocker
for anything.

### CSNOtk / CSNOServ — after Q27
Apache-2.0, so no licensing question. Only worth deploying once you know whether
its API does ECL ([Q27](10-open-questions.md#q27)); if it does, it replaces the
Snowstorm standup entirely.

---

## 5. Order of work

```
TODAY      UTS account request  ·  confirm CDCI on MLDS  ·  ask for the drug master
           (all have lead times and none depend on anything else)

THIS WEEK  DDInter 2.0 → run ddinter/README.md recipe        → Q31, Q32
           Unpack SNOMED → run snomed-releases/README.md     → Q2, Q22
           UNII · NLEM · DrugBank Open Data · ONCHigh        (an afternoon)

NEXT       RxNorm when the UTS account lands                 → Q23
           Check ATC inside DDInter and RxNorm               → §2
           sources.lock committed with every checksum
           Coverage census (P7) — the gate
```

## 6. What "done" looks like

- [ ] `sources.lock` committed, every source with version, SHA-256, licence and
      `redistributable`
- [ ] Every payload in its gitignored directory; nothing licensed in git
- [ ] `row_counts` recorded per source, so the build asserts them and a silent
      upstream change fails loudly
- [ ] Q22, Q23, Q31, Q32 answered from the files
- [ ] ATC sourced without purchase, or the gap consciously accepted
