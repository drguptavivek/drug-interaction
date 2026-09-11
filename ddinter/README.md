# ddinter/

Local-only working directory for DDInter 2.0 downloads. **Nothing in here is
committed.**

## Why the contents are gitignored

DDInter is **CC BY-NC-SA 4.0**. Committing the CSVs to this public repository
would be redistribution — permitted under the licence only with attribution, a
licence notice and a statement of changes, under the same licence. That is what
the release artifact's `NOTICE` and `LICENSE-KB` exist to do
([08 §2](../docs/plan/08-licensing.md#2-the-cc-by-nc-sa-sharealike-obligation--and-why-the-kb-must-not-be-embedded)).
Raw source files in a code repository carry none of that, so they stay out.

Download from **`ddinter2.scbdd.com/download/`** — note the `2`. The older
`ddinter.scbdd.com` is version 1.0 and appears to publish a smaller file set.

## Inspection recipe — answers Q31 and Q32 in ~20 minutes

These are the two remaining source questions, and both are answered by looking
at the files.

### Inventory — which files, how many records ([Q31](../docs/plan/10-open-questions.md#q31))

```bash
cd ddinter
ls -la *.csv
for f in *.csv; do printf '%-45s %8d\n' "$f" "$(( $(wc -l < "$f") - 1 ))"; done
```

- [ ] Are there files for ATC first levels **C, G, J, M, N, S**? The 1.0 site
      published only A, B, D, H, L, P, R, V.
- [ ] Are there separate files for **drug–food**, **drug–disease** and
      **therapeutic duplication**? DDInter 2.0 documents 857 / 8,359 / 6,033
      records respectively ([15](../docs/plan/15-content-types.md)).
- [ ] Do the DDI rows, **after deduplication**, approach the documented
      **302,516**? A pair involving classes A and B appears in *both* files, so
      the raw line-count sum will overshoot.

### Columns — is the mechanism and management text there? ([Q32](../docs/plan/10-open-questions.md#q32))

```bash
head -1 ddinter_downloads_code_A.csv | tr ',' '\n' | nl
head -3 ddinter_downloads_code_A.csv
```

The text **exists** in DDInter — 8,398 distinct mechanism and management
descriptions for DDIs. The question is only whether the bulk CSV ships it, or
just `{DDInterID_A, Drug_A, DDInterID_B, Drug_B, Level}`.

- [ ] Mechanism category present? (absorption / distribution / metabolism /
      excretion / synergy / antagonism / others / unknown)
- [ ] Management or advice text present?
- [ ] Citations present?

**If mechanism category is present**, it is the single most useful field in the
download: it derives the route gate for absorption-mechanism interactions and
sharpens the interruptive tier
([15 §2](../docs/plan/15-content-types.md#2-the-mechanism-taxonomy-is-the-most-valuable-part)).

**If management text is absent**, see [Q32](../docs/plan/10-open-questions.md#q32) —
the preferred answer is to author management text institutionally for the
interruptive tier only, which is a few dozen rules and fully ours to publish.

### Deduplicate and count properly

```bash
# canonicalise each pair as (lo,hi) on DDInter ID, then count distinct
# ADJUST the field numbers to match the header you just printed
awk -F',' 'FNR>1 { a=$1; b=$3; print (a<b ? a"\t"b : b"\t"a) }' *.csv \
  | sort -u | wc -l
```

- [ ] Distinct pair count recorded in `sources.lock` `row_counts`
- [ ] Non-exact duplicates — same pair, different severity across files —
      **reported, not silently resolved** ([04 §3](../docs/plan/04-etl-pipeline.md#3-normalize))

### Coverage against the Indian formulary

The real question, and the Phase 0 gate
([P7](../docs/plan/14-phase-0-runbook.md#p7--coverage-census--the-gate)):

```bash
# distinct drug names in the download
awk -F',' 'FNR>1 {print $2; print $4}' *.csv | sort -u > /tmp/ddinter-drugs.txt
wc -l /tmp/ddinter-drugs.txt      # expect ~2,310 if the download is complete
```

Then match against the 150-molecule stratified sample, and report coverage
**per ATC first level** — an aggregate figure would hide a packaging gap.

## Recording in `sources.lock`

```bash
shasum -a 256 ddinter/*.csv
```

`sources.lock` is committed; the CSVs are not. Record `version`, per-file
`sha256`, `licence: CC-BY-NC-SA-4.0`, `redistributable: true` (subject to
ShareAlike), and the actual `row_counts` — the build asserts them on every
rebuild, so a silent upstream change fails loudly.
