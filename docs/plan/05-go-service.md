# 05 — Go Service Structure

One binary, `ddid`. One companion CLI, `ddictl`. Both built from the same module.

## 1. Package layout

```
cmd/
  ddid/                  main: config, wiring, graceful shutdown
  ddictl/                verify | inspect | diff | sign | dump | bench
internal/
  kbfile/                artifact format: reader, writer, section table, CRC
  kbindex/               in-memory views over the mmap: CSR, code indexes
  resolve/               input code -> ingredient set (SCTID, DDInter, product, salt)
  predicate/             applicability predicate evaluator (route gates, exceptions)
  engine/                pair enumeration, rule lookup, outcome classification
  tiering/               severity + ONCHigh + overlay -> alert tier
  overlay/               overlay file parse, validate, compatibility check
  apirest/               POST /v1/interactions/check and friends
  apicds/                CDS Hooks discovery + order-select + order-sign
  authn/                 offline JWT validation, JWKS cache, fallback credential
  audit/                 structured audit log, no PHI
  obs/                   logging, metrics, health, version
  config/                file + env + flags, validated at startup
pkg/
  ddiapi/                request/response types, published for HMIS integrators
```

`pkg/ddiapi` is the only exported package. Keeping the engine internal is
deliberate: the moment a third party imports `engine`, the artifact format stops
being free to change.

**Dependency direction is strictly downward.** `engine` knows nothing about HTTP;
`apirest` and `apicds` both depend on `engine` and neither depends on the other.
The two API surfaces sharing an engine but not a transport is what makes "same
answers from both surfaces" a compile-time property rather than a test.

## 2. Loading

```go
// internal/kbfile
type KB struct {
    data     []byte      // mmap'd, read-only, MAP_PRIVATE
    sections map[Kind]Section
    meta     Meta
}

func Open(path string, pubkey ed25519.PublicKey) (*KB, error)
```

- `mmap` read-only. Falls back to a plain read into memory on filesystems where
  mmap is unavailable (some NFS-mounted PHC setups) — logged loudly, since RSS
  then equals artifact size.
- Signature verified **before** any section is interpreted. A bad signature is
  fatal, always, with no override flag. This is the one thing that must not be
  configurable: a tampered DDI knowledge base is a patient-harm vector, and an
  override flag is an invitation to use it at 2 a.m.
- Section CRCs verified lazily on first touch, with a `--verify-all-at-start`
  flag for sites that prefer a slower boot and a stronger guarantee.

## 3. In-memory index

Nothing is deserialized into Go objects. The index is a set of typed slice views
over the mapped bytes, built with `unsafe.Slice` on a validated, aligned region.

```go
// internal/kbindex
type Index struct {
    nIngredients uint32
    csrOff  []uint32   // len = n+1
    csrAdj  []uint32   // len = nnz, sorted within each row
    csrRec  []uint32   // len = nnz, parallel to csrAdj
    rules   []Rule     // fixed 16-byte records
    strs    []byte     // string arena

    sctid   []uint64   // sorted; parallel to sctidIng
    sctidIng []uint32
    ddinter  []uint32  // interned DDInter id -> ingredient
    saltToMoiety []uint32
    covered  bitset.Bitset
}
```

### 3.1 Pair lookup

```go
func (ix *Index) Pair(a, b uint32) (Rule, bool) {
    // probe the shorter adjacency row: bounded by min(deg(a), deg(b))
    if ix.deg(a) > ix.deg(b) { a, b = b, a }
    row := ix.csrAdj[ix.csrOff[a]:ix.csrOff[a+1]]
    i, found := slices.BinarySearch(row, b)
    if !found { return Rule{}, false }
    return ix.rules[ix.csrRec[ix.csrOff[a]+uint32(i)]], true
}
```

Properties: zero allocations, no pointers (so no GC scanning of the KB at all,
regardless of size), and the working set is the touched pages only.

**Complexity.** For an n-drug list: n(n−1)/2 probes, each O(log d) where d is the
smaller degree. n=20, d̄≈180 ⇒ 190 × ~8 comparisons ≈ 1,500 comparisons. The
measured cost will be dominated by request parsing and JSON encoding, not by the
lookup — which is the correct place for the cost to be.

**Rejected alternative:** `map[uint64]Rule` keyed on `(lo<<32)|hi`. Simpler to
write, ~12 MB of Go heap for the full set, and every GC cycle walks it. The CSR
costs about a day more to implement and removes the KB from the heap entirely.

### 3.2 Degree skew

A handful of ingredients (warfarin, some azoles) have adjacency rows in the
thousands. The "probe the shorter row" rule handles this: a warfarin-vs-obscure
probe searches the obscure drug's short row. Without that swap, a polypharmacy
list containing two high-degree perpetrators is the worst case; with it, the
worst case is two *medium*-degree drugs, which is bounded and small.

## 4. Resolution

```go
type Resolution struct {
    Input      InputDrug
    Status     ResolutionStatus   // see 06 §3
    Ingredients []IngredientRef   // >1 for an FDC
    Via        ResolutionPath     // direct_substance | via_salt | via_product | via_ddinter
    Notes      []string
}
```

Resolution order, first hit wins, recorded in `Via`:

1. SCTID is a known moiety → direct.
2. SCTID is a known salt → moiety via `IDX_SALT`, `via_salt`, note the salt.
3. SCTID/product code is a product → one or more moieties via `IDX_PROD`,
   `via_product`, route carried forward from the product.
4. DDInter id supplied directly → `via_ddinter`.
5. SCTID present in the pinned SNOMED release but inactive → `stale_code`.
6. Nothing → `unknown_code`.

Route is attached at resolution time because the predicate evaluator needs it,
and because an input that carries no route is a materially weaker input: the
response says so (`route: unknown`) rather than silently assuming systemic.

## 5. Engine and outcome classification

```go
for each unordered pair (i, j) of resolved ingredients, i != j:
    if !covered(i) || !covered(j):
        pair.Status = NotEvaluated                 // -> no_interaction_data
        continue
    rule, ok := index.Pair(i, j)
    if !ok:
        pair.Status = EvaluatedNoInteraction
        continue
    if !predicate.Eval(rule.Pred, ctx{routes, salts, doseForms}):
        pair.Status = EvaluatedNotApplicable       // route gate or exception fired
        continue
    pair.Status = Interaction
    finding := build(rule)
    finding.Tier = tiering.Resolve(rule, overlay)
```

Same-moiety pairs (two products with the same moiety) are reported separately as
`therapeutic_duplication` **only if** such content is actually available — see
[C3](11-challenges-to-the-brief.md#c3). If DDInter turns out not to supply it,
the service can still detect exact moiety duplication structurally, which is
genuinely useful and costs nothing; it must be labelled as structurally derived,
not as sourced knowledge.

## 6. Startup validation

This is where I depart from the brief ([C5](11-challenges-to-the-brief.md#c5)).

The brief asks for "a startup validation that every mapped SCTID still resolves
as active in the loaded India extension release, failing loudly on mismatch."
The check is right. Making it fatal is wrong, for a reason that only shows up in
deployment: the SNOMED release on a machine is upgraded by an administrator who
is not thinking about the DDI service. If drift crashes `ddid`, the hospital
loses interaction checking entirely — and loses it *silently*, because the HMIS
just sees a dead endpoint and carries on prescribing. A service that says
"3 of your 650 drugs have stale codes" is strictly safer than one that says
nothing because it is not running.

Three tiers instead:

| Condition | `ddictl verify` (CI, release gate) | `ddid` at startup |
|---|---|---|
| Signature invalid | fail | **fatal** |
| Section CRC bad | fail | **fatal** |
| Artifact/overlay version incompatible | fail | **fatal** |
| SCTID inactive in the local SNOMED release | **fail the release** | log `ERROR`, mark those ingredients `stale_code`, set `/healthz` to `degraded`, serve |
| SCTID absent entirely | **fail the release** | as above |
| Overlay references an unknown ingredient | fail | log `ERROR`, ignore the entry, serve |
| Release withdrawn (per the local withdrawal list) | fail | log `ERROR`, `degraded`, serve, and stamp every response |

Degraded mode is visible in three places at once: `/healthz`, every response's
`kb.status` field, and a startup banner in the log. A stale code cannot be
invisible, but it also cannot take the hospital's DDI checking offline.

`--strict-terminology` exists for sites that genuinely prefer failure to
degradation. It defaults to **off**, and the choice is recorded in the audit log
at startup so a later incident review can see which posture was configured.

**`ddid` links no terminology client and has no terminology server
configuration key.** This is an explicit non-goal. A terminology server
(Snowstorm or similar) is a build- and curation-time dependency only — see
[12 Part A](12-terminology-tooling.md#part-a--terminology-server). "We already
run Snowstorm, just call it for the startup check" is a reasonable-sounding
suggestion that would silently destroy the offline property, and it is easiest to
refuse by having nowhere to configure it.

Withdrawal without network access is handled by a `withdrawn.txt` file shipped
with each release listing superseded-and-withdrawn version strings; sites that
never update never learn of a withdrawal, which is an honest limitation of an
offline system and should be stated in the deployment guide rather than papered
over.

## 7. Configuration and overlay loading

Precedence: defaults → config file → environment → flags.

```yaml
kb:
  path: /var/lib/ddi/kb.ddi
  signature: /var/lib/ddi/kb.ddi.sig
  strict_terminology: false
overlay:
  path: /etc/ddi/overlay.yaml
snomed:
  release_manifest: /var/lib/snomed/release.json   # for the startup check
auth:
  mode: oidc            # oidc | static | none
  issuer: https://keycloak.aiims.internal/realms/hospital
  jwks_cache: /var/lib/ddi/jwks.json
  jwks_max_age: 720h    # offline grace
  audience: ddi-service
server:
  listen: 127.0.0.1:8443
  tls: { cert: ..., key: ... }
audit:
  path: /var/log/ddi/audit.jsonl
  include_drug_codes: true
  include_patient_id: false        # cannot be enabled; see below
```

Overlay validation at load: `kb_compatibility` must match the loaded KB's version
string, every referenced `public_id` must exist, every suppression must carry a
rationale and an approver. A structurally invalid overlay is fatal — unlike
terminology drift, a bad overlay is a *configuration* error the administrator can
fix immediately, and serving with a half-applied suppression list is worse than
not serving.

`include_patient_id` is present in the schema and rejected if set to `true`. The
service never receives or logs patient identifiers: requests carry coded
medication lists and optional coarse context (age band, renal band), and that is
all. This keeps the service outside the scope of most DPDP Act 2023 obligations
and makes the audit log safe to ship to a central quality team.

## 8. Auth ([C8](11-challenges-to-the-brief.md#c8))

Keycloak as specified — but Keycloak is a network dependency, and the brief also
says no network at query time. Reconciliation:

- **Offline JWT validation.** JWKS fetched when reachable, cached to disk,
  signature validated locally. No introspection call on the request path. A JWKS
  older than `jwks_max_age` logs a warning and continues; expiry of the *cache*
  must not deny clinical care.
- **`mode: static`** for disconnected sites: a per-site credential in a file,
  rotated by the same USB stick that delivers KB updates. Less good than OIDC,
  and honest about it.
- **`mode: none`** permitted only on loopback, and refused if `server.listen` is
  not a loopback address. A PHC running the service as a sidecar to a single
  HMIS process on the same machine is the realistic deployment, and forcing token
  plumbing there buys nothing.

## 9. Observability

Prometheus metrics, all label-bounded (no drug codes in labels — that is a
cardinality bomb and arguably a confidentiality one):

| Metric | Type | Why it matters |
|---|---|---|
| `ddi_check_duration_seconds` | histogram | latency SLO |
| `ddi_findings_total{tier}` | counter | alert burden |
| `ddi_unresolved_total{status}` | counter | **the coverage canary** |
| `ddi_pairs_not_evaluated_total` | counter | how often we are answering "don't know" |
| `ddi_kb_status{version,status}` | gauge | degraded-mode visibility |
| `ddi_overlay_suppressed_total` | counter | is the overlay hiding too much |

`ddi_unresolved_total` rising is the earliest available signal that the formulary
has moved past the KB — the practical replacement for the continuous monitoring
the brief declines to build.

## 10. Testing

| Layer | Approach |
|---|---|
| `kbfile` | Round-trip property tests; fuzz the reader against corrupted bytes (a truncated USB copy is a realistic input, and the reader must not panic on it) |
| `kbindex` | Differential test: CSR lookup vs a naive map over the same data, on random KBs |
| `predicate` | Table-driven over every node type; a predicate that fails to evaluate is treated as "does not fire", and that default is tested explicitly |
| `engine` | Golden files per outcome-taxonomy case |
| `apirest`/`apicds` | Contract tests from the OpenAPI and CDS Hooks schemas; **plus a test asserting both surfaces return the same findings for the same input** |
| System | The departmental validation set, run against the built artifact in CI |
| Performance | `go test -bench` with an allocation assertion (`0 allocs/op`) on the pair path |
