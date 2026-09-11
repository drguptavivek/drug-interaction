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
