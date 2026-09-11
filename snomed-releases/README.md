# snomed-releases/

Local-only working directory for SNOMED CT release archives. **Nothing in here
is committed** — see the rules below.

## Why the contents are gitignored

This repository is **public**. SNOMED CT content (concept descriptions, FSNs,
relationships, reference sets) is licensed, and distribution to non-affiliates is
restricted. Committing an RF2 archive — or extracted `sct2_*.txt` /
`der2_*.txt` files — to a public repository would breach the SNOMED Affiliate
Licence and the NRCeS national licence terms.

So the `.gitignore` entries covering this folder are a **compliance control**,
not housekeeping. If a build fails because a release file is missing, the fix is
to fetch it locally, never to commit it. The same applies to DDInter, RxNorm/UMLS
and CredibleMeds — see [`docs/plan/08-licensing.md`](../docs/plan/08-licensing.md).

## Expected layout

One directory per pinned release, named `<source>_<version>`:

```
snomed-releases/
├── .gitkeep                        (tracked — keeps the folder in the repo)
├── README.md                       (tracked — this file)
├── SnomedCT_InternationalRF2_PRODUCTION_<yyyymmdd>T120000Z/
│   └── Snapshot/                   ← Snapshot only; Full is not needed
│       ├── Terminology/
│       │   ├── sct2_Concept_Snapshot_INT_<yyyymmdd>.txt
│       │   ├── sct2_Description_Snapshot-en_INT_<yyyymmdd>.txt
│       │   └── sct2_Relationship_Snapshot_INT_<yyyymmdd>.txt
│       └── Refset/
│           ├── Content/            ← language, simple, association refsets
│           ├── Map/                ← simple map refsets (UNII/ATC if present)
│           └── Metadata/
└── SnomedCT_IndiaExtensionRF2_PRODUCTION_<yyyymmdd>T120000Z/
    └── Snapshot/ …
```

The India extension has a **dependency** on a specific International Release
edition. Both must be present, and the pair must be consistent — the extension's
`der2_ssRefset_ModuleDependency` rows state which International version it was
built against. Loading a mismatched pair produces dangling references that
surface much later as unresolvable concepts.

## What the plan expects from this folder

| Consumer | Needs | Reference |
|---|---|---|
| ETL `acquire` stage | The archives, verified against `sources.lock` SHA-256 | [04 §2](../docs/plan/04-etl-pipeline.md#2-acquire--source-pinning) |
| ETL `stage`/`normalize` | Snapshot `Terminology/` and `Refset/` files → PostgreSQL | [04 §3](../docs/plan/04-etl-pipeline.md#3-normalize) |
| Snowstorm import | The same archives, imported into an ephemeral pinned container | [12 §A4](../docs/plan/12-terminology-tooling.md#a4-reproducibility--the-constraint-that-shapes-the-integration) |
| `IDX_HIST` build | Historical association refsets (`SAME AS`, `REPLACED BY`, `POSSIBLY EQUIVALENT TO`) | [13 §3](../docs/plan/13-hmis-neutral-integration.md#3-release-skew--the-hmiss-codes-will-go-stale-too) |
| `ddid` startup validation | A local release manifest (not these archives, and **not** a terminology server) | [05 §6](../docs/plan/05-go-service.md#6-startup-validation) |

## Recording a release in `sources.lock`

`sources.lock` **is** committed — it holds versions and checksums, not content.
That is what makes the build reproducible without publishing licensed data.

```bash
# from the repo root
shasum -a 256 snomed-releases/SnomedCT_*.zip
```

Then add or update the matching entry in `sources.lock` with `version`, `sha256`,
`licence` and `redistributable: false`.

## Inspection recipe — answers three open questions in ~20 minutes

These run against unpacked RF2 on your laptop. They assume standard RF2 column
order; sanity-check with `head -1 <file>` first, since a wrong field index gives
a confidently wrong count rather than an error.

Set a base path once:

```bash
INTL=snomed-releases/SnomedCT_InternationalRF2_.../Snapshot
CDCI=snomed-releases/<CDCI package>/Snapshot
```

### Q22 — is there a UNII map reference set?

The one that decides how load-bearing RxNorm is
([12 §B1](../docs/plan/12-terminology-tooling.md#b2-what-dropping-rxnorm-actually-costs)).

```bash
ls "$INTL"/Refset/Map/                                   # what map refsets ship at all
grep -i "unii" "$INTL"/Terminology/sct2_Description_Snapshot-en_*.txt | head
# refset ids actually populated in the simple map file (field 5 = refsetId):
cut -f5 "$INTL"/Refset/Map/der2_sRefset_SimpleMapSnapshot_*.txt | sort -u
# then look each id up to see what it is:
grep -F -f <(cut -f5 "$INTL"/Refset/Map/der2_sRefset_SimpleMapSnapshot_*.txt | sort -u) \
     "$INTL"/Terminology/sct2_Description_Snapshot-en_*.txt | cut -f5,8 | sort -u
```

A UNII map present → the SNOMED-side UNII anchor derives natively. Absent → it
must come via RxNorm (`SCTID → RXCUI → UNII`), which makes RxNorm structurally
necessary rather than optional.

### Q2 / Q4 — what is in the India (CDCI) package?

Semantic-tag distribution over active FSNs. Fields: `$3` active, `$7` typeId
(`900000000000003001` = FSN), `$8` term.

```bash
awk -F'\t' '$3==1 && $7=="900000000000003001" {
  if (match($8, /\([^)]*\)$/)) print substr($8, RSTART+1, RLENGTH-2)
}' "$CDCI"/Terminology/sct2_Description_Snapshot*.txt | sort | uniq -c | sort -rn
```

Expect `(medicinal product)`, `(clinical drug)`, `(real clinical drug)`,
`(product)` — and **few or no `(substance)` rows**. That is the confirmation that
the extension supplies the product layer while the International Release supplies
substances, which is the good outcome for the mapping design.

Same command against `$INTL` gives the substance count — the size of spoke A's
target space.

### Module dependency — do the two packages actually pair?

```bash
cat "$CDCI"/Refset/Metadata/der2_ssRefset_ModuleDependencySnapshot*.txt | column -t -s$'\t'
```

`sourceEffectiveTime` / `targetEffectiveTime` state which International version
the extension was built against. A mismatched pair loads with dangling
references that surface much later as unresolvable concepts.

### Relationship counts the design depends on

Fields: `$3` active, `$8` typeId.

```bash
for t in 738774007:"Is modification of" \
         127489000:"Has active ingredient" \
         762949000:"Has precise active ingredient" \
         732943007:"Has basis of strength substance"; do
  id=${t%%:*}; name=${t#*:}
  n=$(awk -F'\t' -v id="$id" '$3==1 && $8==id' \
        "$INTL"/Terminology/sct2_Relationship_Snapshot*.txt \
        "$CDCI"/Terminology/sct2_Relationship_Snapshot*.txt 2>/dev/null | wc -l)
  printf '%-38s %s\n' "$name" "$n"
done
```

`Is modification of` gives the size of the salt/ester/prodrug candidate set that
Phase 1 must classify ([04 §3.1](../docs/plan/04-etl-pipeline.md#31-substance-graph-and-closure)).
The three ingredient relationships confirm product decomposition works as
designed — if `Has active ingredient` is near zero in the CDCI package, the FDC
decomposition assumption needs revisiting.

### Combi packs

CDCI excludes them. Worth sizing the gap against the drug master locally, since
every combi-pack row needs decomposing into component products
([13 §4.3](../docs/plan/13-hmis-neutral-integration.md#43-drug-master-coding-qa-report)).

---

## Phase 0 questions this folder will answer

Two open questions can be settled just by unpacking a release and looking
([`docs/plan/10-open-questions.md`](../docs/plan/10-open-questions.md)):

- **Q2** — does the India edition contain substance concepts of its own, or only
  products and dose forms layered on the International Release?
- **Q22** — does the release ship a **UNII map reference set**? If not, the
  SNOMED-side UNII anchor has to be derived via RxNorm, which makes RxNorm
  structurally necessary rather than optional.

Both are `grep` questions against the unpacked `Refset/Map/` and
`Terminology/sct2_Concept` files. Worth doing early — Q22 in particular changes
how much of the design leans on RxNorm.
