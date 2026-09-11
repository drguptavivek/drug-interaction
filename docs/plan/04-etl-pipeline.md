# 04 — ETL Pipeline and Artifact Build

Python, run in CI, entirely separate from the Go runtime. The output is one
signed file plus a manifest. The pipeline is a build system, not a service: it
has no uptime requirement, and it is allowed to be slow.

## 1. Stage map

```
 acquire ──► stage ──► normalize ──► link ──► candidates ──► [HUMAN CURATION] ──►
                                                                             │
             ┌───────────────────────────────────────────────────────────────┘
             ▼
          compose ──► build ──► verify ──► sign ──► package
```

| Stage | Input | Output | Idempotent | Determinism guarantee |
|---|---|---|---|---|
| `acquire` | pinned `sources.lock` | raw files + SHA-256 | yes | checksum match or hard fail |
| `stage` | raw files | `raw.*` tables, untouched content | yes | row counts asserted |
| `normalize` | `raw.*` | `stg.*`, typed, deduplicated | yes | pure function of `raw.*` |
| `link` | `stg.*` | anchor tables, substance graph, closures | yes | pure |
| `candidates` | `stg.*` + department submissions | ranked candidate sets | yes | `tool_version` recorded |
| `compose` | approved proposals | `mapping_projection`, applicable rules | yes | pure function of the append-only log |
| `build` | composed tables | `kb-<version>.ddi` | yes | byte-identical for the same inputs |
| `verify` | artifact | pass/fail report | yes | |
| `sign` | artifact | detached signature | no (key access) | |
| `package` | artifact + NOTICE + manifest | release tarball | yes | |

Reproducibility is a hard requirement, not an aspiration: `SOURCE_DATE_EPOCH`
pinned, all dictionary iteration sorted, no `hash()`-order dependence, no
wall-clock in the artifact body (build time lives in the manifest, outside the
hashed region). Phase 4 exit requires two independent builders to produce the
same `artifact_sha256`.

## 2. `acquire` — source pinning

```yaml
# sources.lock  (hand-edited, reviewed, committed)
- source: DDINTER
  version: "2.0"
  url: "<retrieved in Phase 0>"
  sha256: "…"
  licence: CC-BY-NC-SA-4.0
  redistributable: true            # subject to ShareAlike, see 08
- source: SNOMED_INT
  version: "20250401"
  sha256: "…"
  licence: SNOMED-AFFILIATE
  redistributable: false           # descriptions may not be republished
- source: SNOMED_IN
  version: "20250401"
  sha256: "…"
  licence: SNOMED-IN-NATIONAL
  redistributable: false
- source: UNII
  version: "2025-03"
  sha256: "…"
  licence: PUBLIC-DOMAIN
  redistributable: true
- source: OPENFDA
  version: "2025-03"
  licence: PUBLIC-DOMAIN
  redistributable: true
- source: ONCHIGH
  version: "2016"
  licence: <Phase 0 finding>
  redistributable: <Phase 0 finding>
- source: CREDIBLEMEDS
  version: "…"
  licence: CREDIBLEMEDS-TOU
  redistributable: false           # NOT shipped; site-supplied overlay only
- source: DRUGBANK_OPEN
  version: "5.1"
  licence: CC0-1.0
  redistributable: true            # vocabulary subset ONLY, not the full academic set
```

`acquire` refuses to run without a checksum. A source whose checksum changed
upstream is a **release-blocking event** requiring a human to re-read the licence
and re-run the Phase 0 assertions — this is the cheap substitute for the
continuous monitoring the brief (reasonably) declines to build, and it is the
mechanism referenced in [C6](11-challenges-to-the-brief.md#c6).

## 3. `normalize`

| Source | Normalization notes |
|---|---|
| DDInter | Split the drug list from the pair list. **Assert** the drug and pair counts and record them; the brief's ~2,310 / ~302,000 are hypotheses (see [C3](11-challenges-to-the-brief.md#c3)). Canonicalise each pair to `(lo, hi)` by DDInter ID, drop exact duplicates, report non-exact duplicates (same pair, different severity) rather than silently picking one. |
| SNOMED RF2 | Snapshot only, not Full. Load `concept`, `description`, `relationship`, `sct2_RelationshipConcreteValues`, plus the UNII/ATC simple map refsets if present. Keep `active=0` rows — inactivity is a fact the service needs, not noise to filter. |
| CDC-India | Product codes with composition. Retain original strings. |
| UNII / GSRS | Preferred substance name, all synonyms, and the salt→parent relationship. The latter is what allows UNII comparison at a consistent level (see [03 §5](03-candidate-ranking.md#5-anchor-agreement-semantics)). |
| WHO ATC | Full index to level 5; retain **all** codes per substance. |
| openFDA | Extract only the label sections plausibly relevant: `drug_interactions`, `contraindications`, `warnings`. Used as corroborating evidence displayed to curators, **not** as a rule source — free-text label mining produces rules nobody can defend at a mortality review. |
| ONCHigh | ~15 drug-class pairs. Expand class→member using SNOMED substance descendants, then review the expansion by hand. Class expansion is where "always alert" lists quietly become 400 alerts. |

### 3.1 Substance graph and closure

```
substances      := descendants_of(105590001 |Substance|)
modification    := { (a,b) : a -[738774007 Is modification of]-> b }
```

Cycle detection on `modification` is mandatory and failures are reported, never
auto-broken. The transitive closure is computed but **not used for automatic
collapse** — it produces the candidate set only, and every edge is classified:

| `derivation_kind` | Detection heuristic | Default proposal |
|---|---|---|
| `salt` | Child FSN = parent FSN + a known counter-ion token (`sodium`, `hydrochloride`, `sulfate`, `maleate`, `besilate`, …) and UNII parent-relationship confirms | collapse |
| `ester` | FSN contains `acetate`/`propionate`/`palmitate`/`decanoate`/… **and** the parent is not a counter-ion pattern | **no_collapse**, review |
| `prodrug` | ATC-5 differs from parent, or an openFDA/DrugBank-Open flag, or a curated prodrug list | **no_collapse**, review |
| `complex` | `ferric carboxymaltose`, `iron sucrose`, polyvalent-cation patterns | flagged, review |
| `unknown` | anything else | blocked from band A; hard queue |

This table is the operational form of [C4](11-challenges-to-the-brief.md#c4). The
heuristics are imperfect *by design* — their job is to route work to the right
queue, not to decide.

### 3.2 Product decomposition

```
product -[127489000 has active ingredient]->          moiety-level substance
product -[762949000 has precise active ingredient]->  salt-level substance
product -[732943007 has basis of strength substance]-> substance the strength refers to
```

Arity check: the number of distinct `has active ingredient` targets must equal
the FDC component count parsed from the name. Mismatch ⇒ human review. Indian
FDCs are frequently 3- and 4-component, and a silently dropped component is a
silently missed interaction.

## 4. `compose`

Pure function of the approved proposals:

1. Resolve `merged_into` chains to surviving ingredients.
2. Project approved spokes → `mapping_projection`.
3. Restrict `interaction_rule` to pairs where **both** ingredients are mapped
   and active.
4. Attach `applicability` predicates from approved `exception_rule` rows.
5. Compute `ingredient_coverage` per ingredient from the DDInter drug list —
   `is_covered` is true iff the DDInter side knows the drug at all.
6. Intern all prose into `text_block`.
7. Compute the global default tiering (institutional overlays apply later, at
   runtime).

## 5. `build` — sizing

Working estimates at phase-1 scope (650 ingredients), with the full-DDInter
figure alongside because the artifact may be built either way:

| Section | Phase 1 (650 moieties) | Full (2,310 drugs) |
|---|---|---|
| Ingredients | 650 × 64 B ≈ 42 KB | 2,310 × 64 B ≈ 148 KB |
| Pair CSR (keys + offsets) | ~120 k pairs × 16 B ≈ 1.9 MB | ~302 k × 16 B ≈ 4.8 MB |
| Rule records | ~120 k × 24 B ≈ 2.9 MB | ~7.3 MB |
| Interned prose | ~8 MB | ~25 MB |
| Code indexes (SCTID, DDInter, product) | ~1 MB | ~3 MB |
| **Total** | **~14 MB** | **~40 MB** |

Compressed for distribution, uncompressed on disk for mmap. This is small enough
that the "no network at query time" constraint costs nothing, and small enough to
ship to a PHC on a USB stick — which, realistically, is how it will get there.

## 6. Artifact format

A flat, little-endian, mmap-able file. Not `go:embed`
([C1](11-challenges-to-the-brief.md#c1)), not SQLite.

```
offset  size   field
0       8      magic            "DDIKB\0\0\1"
8       4      format_version   uint32
12      4      section_count    uint32
16      32     manifest_sha256  — hash of the manifest this artifact was built with
48      N×24   section table    { kind u32, offset u64, length u64, crc32 u32 }
...            sections, each 64-byte aligned
```

| Section kind | Contents |
|---|---|
| `META` | KB version string, source release list, build inputs, licence NOTICE text |
| `STR` | One byte blob; all strings are `(offset u32, len u32)` into it |
| `ING` | `{ public_id_str, name_str, flags, coverage_bits }`, dense `uint32` ids |
| `CSR_OFF` | `uint32[n_ingredients+1]` — row offsets |
| `CSR_ADJ` | `uint32[nnz]` — neighbour ingredient ids, sorted within each row |
| `CSR_REC` | `uint32[nnz]` — index into `RULE` |
| `RULE` | `{ severity u8, tier_default u8, flags u16, mech u32, mgmt u32, pred u32 }` |
| `PRED` | Serialized applicability predicates (route gates, exceptions) |
| `IDX_SCT` | sorted `uint64 sctid → uint32 ingredient` |
| `IDX_DDI` | sorted DDInter id → ingredient |
| `IDX_PROD` | product code → ingredient list (offsets into `PROD_ING`) |
| `IDX_SALT` | salt sctid → moiety ingredient |
| `ONCH` | ONCHigh membership bitmap over pairs |
| `SIG` | detached signature is a **separate file**; this section holds the public key id only |

Design choices worth defending:

- **CSR, not a hash map.** A pair probe is a binary search within the shorter of
  the two adjacency rows. For a 20-drug list that is 190 binary searches over
  rows averaging ~180 entries — roughly 8 comparisons each, all in cache, zero
  allocations. A `map[uint64]uint32` with 302 k entries costs ~12 MB of pointer-ful
  Go heap and puts every lookup in the GC's way for no latency benefit.
- **Interned strings.** DDInter management text is highly repetitive; interning
  reduces the prose section by roughly an order of magnitude and makes the
  artifact diffable.
- **CRC per section.** Detects a truncated USB-stick copy immediately rather
  than at the first query that touches the damaged region.
- **No compression inside the file.** mmap + page cache beats decompress-on-load
  for a read-mostly workload, and keeps RSS proportional to what is actually
  touched — which matters on a PHC machine with 2 GB of RAM.

### 6.1 Why not SQLite

Considered seriously, and rejected for the *runtime* path only.

| | Custom mmap | SQLite read-only |
|---|---|---|
| Pair probe latency | ~100 ns | ~2–5 µs |
| Allocations per query | 0 | non-zero (cgo or pure-Go driver) |
| Dependencies | none | cgo, or a pure-Go engine |
| Ad-hoc inspection | needs `ddictl` | `sqlite3` |
| Implementation cost | ~2 eng-weeks | ~0.5 eng-weeks |

SQLite is the better choice if latency budget were 50 ms. At 5 ms p99 with an
in-process index and a zero-dependency single binary as stated constraints, the
custom format wins. **SQLite is still used for the curation-side export** and for
`ddictl dump`, so field inspection is easy.

## 7. `verify`

Runs before signing; failure blocks the release.

| Check | Assertion |
|---|---|
| Structural | every section CRC valid; offsets in range; CSR symmetric |
| Referential | every `CSR_ADJ` id < `n_ingredients`; every string ref in range |
| Terminology | every SCTID in `IDX_SCT` is present **and active** in the pinned SNOMED release |
| Licensing | no row traceable to a `redistributable = false` source |
| Coverage | `ingredient_coverage` present for every ingredient |
| Validation set | ≥ 90% of department validation cases produce the expected outcome; every miss listed |
| Alert volume | replay of shadow-mode order logs yields ≤ 2 interruptive alerts per 100 orders |
| Determinism | second build from the same manifest produces an identical hash |
| Diff sanity | release diff contains no unexplained `retire` of an ingredient that appeared in a prior release |

The licensing check is worth emphasising: it is a **build-time gate** derived
mechanically from `source_release.redistributable`. Compliance that depends on
someone remembering is compliance that fails.

## 8. `sign` and `package`

Ed25519 detached signature over `artifact_sha256`. Private key in an
institutional HSM or, failing that, an offline machine — the pragmatic answer for
AIIMS is likely a hardware token held by the informatics lead. Public key is
compiled into the binary (a key is not licensed content, so this raises none of
the [C1](11-challenges-to-the-brief.md#c1) problems).

Release tarball:

```
ddi-kb-ddinter2.0+snomedIN20250401+r7.tar.zst
├── kb.ddi
├── kb.ddi.sig
├── manifest.json        # sources, versions, checksums, build inputs, tool versions
├── NOTICE               # verbatim attributions, licence texts, ShareAlike notice
├── LICENSE-KB           # CC BY-NC-SA 4.0 as applied to this artifact
└── diff-r6-to-r7.md     # human-readable, clinically annotated
```

`NOTICE` and `LICENSE-KB` are inside the tarball *and* mirrored in the `META`
section, so an artifact separated from its tarball still carries its own licence.
That is a ShareAlike requirement, not a nicety — see [08](08-licensing.md).
