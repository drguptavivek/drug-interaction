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
| 4 | **GSRS / UNII** (NCATS) | UNII anchor, **ACTIVE MOIETY** relationship — the salt-collapse cross-check | none | ❌ **now the key one** |
| 5 | **ATC** | Anchor, tiering | *see §2 — do not buy* | ❌ |
| 6 | **NLEM 2022 (India)** | Indian-market prior (F7); PHC/CHC overlay | none | ❌ |
| 7 | **ONCHigh** | The "always alert" tier | none | ❌ |
| 8 | **DrugBank Open Data** | Name/synonym vocabulary only | none | ❌ |
| ~~9~~ | ~~**openFDA labels**~~ | — | — | ❌ **dropped** |
| 10 | **CredibleMeds** | TdP overlay — **never shipped** | registration | optional |
| 11 | **AIIMS drug master** | [Q24](10-open-questions.md#q24); the coding exercise | institutional | ❌ **ask now** |
| 12 | **BHTS / CSNOServ** | Terminology service — **is Snowstorm**, Indian extensions pre-integrated | none | hosted; local deploy optional |

---

## 1. Start these today — they have lead times

### Q22 — settle it this week, it is now load-bearing

Not a download, but it belongs at the top. With RxNorm dropped there is **no
fallback** for deriving UNII on the SNOMED side: either the release ships a UNII
map refset, or that half of the third anchor becomes name-derived and the anchor
check weakens ([12 §B2](12-terminology-tooling.md#b2-what-dropping-rxnorm-actually-costs)).

It is a `grep` against files already on the laptop —
[`snomed-releases/README.md`](../../snomed-releases/README.md#q22--is-there-a-unii-map-reference-set).

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
| **DDInter** — assigns ATC codes to its drugs, and updated them in 2.0 | Every drug in the interaction set — exactly the set needing the anchor |
| **SNOMED** — an ATC map refset, if the release ships one | Check alongside [Q22](10-open-questions.md#q22), same `Refset/Map/` listing |

- [ ] Confirm ATC codes are present in the DDInter download
- [ ] Check for an ATC map refset while answering Q22
- [ ] Only if both fail, reconsider — and note that ATC is an *advisory* anchor
      compared with UNII ([11 C10](11-challenges-to-the-brief.md#c10)), so a gap
      here is not fatal. With RxNorm dropped, DDInter is the primary ATC route

Remember ATC is **multi-valued** per moiety. Whichever route supplies it, retain
**all** codes per substance and compare by set overlap, never equality.

---

## 3. Free, no account, fetch when convenient

### GSRS / UNII — now the most important free source
Public domain, from NCATS. GSRS is the system that *issues* UNIIs, so it is the
substance authority rather than a relay.

**Take the data export for the build.** A running server is not a pinned file,
and the build must stay reproducible
([12 §B3](12-terminology-tooling.md#b3-gsrs-deploy-or-just-take-the-data)).
Deployable software exists at `ncats/gsrs3-main-deployment` — deploy it only if
curators want structure search and synonym exploration during Phase 3.

- [ ] UNII substance export downloaded and checksummed
- [ ] **Substance relationships included, not just the flat UNII code list.**
      This is the part that is easy to miss and expensive later: the
      **ACTIVE MOIETY** relationship is what lets UNII be compared at a
      consistent level on both spokes, and it is now also the independent
      cross-check on salt-versus-prodrug collapse that RxNorm's IN/PIN was going
      to provide ([12 §B1](12-terminology-tooling.md#b1-why-rxnorm-was-proposed-and-why-gsrs-covers-it))
- [ ] Names and synonyms included — they feed retriever R5
- [ ] Local GSRS deployment: **optional**, Phase 3, only on curator request

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

### openFDA — dropped
Several GB of label JSON for something already restricted to *corroborating
evidence shown to curators*, never a rule source. Out of scope. If a curator ever
asks for label text on a specific molecule, they can look it up online; that does
not justify ingesting the corpus.

### CredibleMeds — optional, and never shipped
Register only if you want the TdP overlay at AIIMS. It is **site-supplied under
your own registration**, converted locally by `ddictl`, and never enters the
build ([11 C2](11-challenges-to-the-brief.md#c2)). Registering is not a blocker
for anything.

### BHTS / CSNOServ — hosted first
**CSNOServ is Snowstorm**, so ECL is available and the Indian extensions are
already loaded. Use **BHTS hosted** for interactive curation and the ECL
expansion cache for the build; that way there is no Elasticsearch stack for us to
operate ([12 §A3](12-terminology-tooling.md#a3-which-server--settled)).
[Q27](10-open-questions.md#q27) narrows to the practical question of endpoint
shape, which the ECL client needs anyway.

---

## 5. Order of work

```
TODAY      confirm CDCI on MLDS  ·  ask for the AIIMS drug master
           (both have lead times and neither depends on anything else)

THIS WEEK  Unpack SNOMED → run snomed-releases/README.md     → Q2, Q22  ← do first
           DDInter 2.0   → run ddinter/README.md recipe      → Q31, Q32
           GSRS export (with relationships) · NLEM · DrugBank Open Data · ONCHigh
           Check ATC inside DDInter, and for an ATC refset   → §2

NEXT       sources.lock committed with every checksum
           Coverage census (P7) — the gate
```

Q22 moved to the front of the week: with RxNorm dropped it has no fallback, and
it changes the weight of the UNII anchor in the ranker.

## 6. What "done" looks like

- [ ] `sources.lock` committed, every source with version, SHA-256, licence and
      `redistributable`
- [ ] Every payload in its gitignored directory; nothing licensed in git
- [ ] `row_counts` recorded per source, so the build asserts them and a silent
      upstream change fails loudly
- [ ] Q22, Q31, Q32 answered from the files
- [ ] ATC sourced from DDInter without purchase, or the gap consciously accepted
- [ ] GSRS export includes substance **relationships**, not just codes
