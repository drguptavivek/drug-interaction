# 13 — HMIS-Neutral Integration

Two decisions taken after the first draft:

1. The system must be **HMIS-neutral** — no vendor- or site-specific integration
   code in our scope.
2. **SCTIDs will be added to the HMIS drug master**, so the calling system sends
   SNOMED codes.

Both are good decisions. Together they remove the largest variance in the plan
(the HMIS adapter, [Q1](10-open-questions.md#q1), 2–6 eng-weeks and unknowable
until month 5). But they do not remove the work — they **move it across the
boundary**, from an integration project we would have run to a drug master
coding project the institution will run. That project needs tooling, governance
and a quality gate, and none of those existed in the previous draft.

This document covers what changes.

---

## 1. The contract becomes the whole integration

Neutral means the published contract is the only thing we ship, and everything
site-specific lives on the other side of it.

| Previously | Now |
|---|---|
| `ddi-hmis-shim` adapting the AIIMS HMIS protocol — a project deliverable | **Removed from scope.** If a site needs a shim, the site builds it |
| Protocol discovery deferred to Phase 5 | No protocol discovery needed |
| Phase 5 blocked on an HMIS test instance | Phase 5 blocked on nothing outside our control |
| Integration correctness verified by us, per site | Verified by the site, against a **conformance kit** we publish |

What "neutral" forbids, stated so it can be enforced in review:

- No HMIS-, vendor- or site-specific branching in the binary. Site variation is
  expressed **only** through the overlay file, which is data.
- No assumption of co-location, of a particular auth mode, or of the caller
  holding state. Every call is stateless, idempotent, and carries everything it
  needs.
- No presentation decisions in the payload. Severity and tier are **codes**, not
  colours, icons or HTML. The HMIS decides how to render. A `detail` string is
  Markdown, and the contract says so, so a plain-text renderer degrades sanely.
- No callbacks, no webhooks, no requirement that the HMIS expose an endpoint.

## 2. The crux: *which* SCTIDs will the HMIS hold?

This is the question that decides whether the decision simplifies things or
quietly creates a new failure mode. An HMIS drug master row looks like this:

```
DM-104422 | PANTOCID 40MG TAB | Pantoprazole 40 mg | tablet | oral
DM-110877 | AUGMENTIN 625 TAB | Amoxicillin 500 mg + Clavulanic acid 125 mg | tablet | oral
```

"Add the SCTID" can mean three different things, and they are not equivalent:

| Option | What is coded | Availability for Indian products | Consequence for us |
|---|---|---|---|
| **A — substance SCTIDs** | One substance code per active ingredient (`395821009 \|Pantoprazole\|`) | **High.** Substances are International Release core content and stable | Simplest and most robust. But an FDC row needs *several* codes, and route/form must come from the HMIS's own fields |
| **B — product SCTIDs** | One product concept per drug master row | **Partial, and better than first assumed.** CDCI *does* supply branded (RCD) and generic (CD) medicine concepts in RF2 ([08 §3.4](08-licensing.md#34-common-drug-codes-for-india-cdci--what-it-actually-is)) — but branded coverage is being built out incrementally and is targeted at national programmes | Ideal when available: strength, form and route come with the code. Covers part of a tertiary formulary, not all of it |
| **C — CDCI codes** | The ABDM-sanctioned Indian medicine code | Same as B — CDCI *is* the SNOMED India drug extension, so B and C are one option | Equivalent to B; decomposed via the same product tables |

**Realistic expectation: mostly A, some B/C where available, and a long tail of
rows that resolve to neither.** Planning for A as the common case and treating
B/C as an upgrade is the safe posture.

Two consequences follow, and both need contract changes.

### 2.1 Route and dose form must be inputs, not inferences

Under option A, the substance code carries no route. But route gates
applicability — that was [C11](11-challenges-to-the-brief.md#c11), and it is one
of the larger alert-fatigue levers in the design. Oral iron chelates
levothyroxine; IV iron does not. Topical ketoconazole should not fire CYP3A4
alerts; ophthalmic timolol *should* fire beta-blocker alerts.

So `route` becomes a **first-class request field that the HMIS is expected to
populate from its own drug master**, not something we derive from the code. The
HMIS already has this field — it is on the drug master row and usually on the
order — so this costs the integrator nothing and is far more reliable than
inferring it.

Where route is absent, the response says `route: unknown` and the service applies
the systemic × systemic default, flagging every finding that depended on that
assumption. Silent assumption is the thing to avoid, not the assumption itself.

### 2.2 One prescribed item may carry several ingredient codes

India is FDC-heavy ([R5](09-risks.md)). Under option A, `AUGMENTIN 625 TAB` is
one prescribed item with two substance SCTIDs. The current request model's
`coding[]` array means "alternative codings of the same concept" — which is a
different thing, and conflating the two would silently turn one FDC into two
independent orders, or worse, treat two ingredients as competing codings and
discard one.

The request model therefore separates them:

```jsonc
{
  "ref": "2",
  "local_id": "DM-110877",                  // HMIS drug master row — REQUIRED
  "display": "AUGMENTIN 625 TAB",           // for echo only, never parsed
  "route": "systemic_oral",
  "dose_form": "tablet",

  "coding": [                               // codings of the PRODUCT (option B/C)
    { "system": "urn:cdc-india", "code": "CDC-IN-10511" }
  ],

  "ingredients": [                          // substance codings (option A)
    { "coding": [{ "system": "http://snomed.info/sct", "code": "372687004" }],
      "strength": { "value": 500, "unit": "mg" } },
    { "coding": [{ "system": "http://snomed.info/sct", "code": "96068000" }],
      "strength": { "value": 125, "unit": "mg" } }
  ]
}
```

Resolution precedence: `coding` (product) if it resolves, else `ingredients`,
with `via` reporting which path was taken. An HMIS supplying **both** gets a free
cross-check — if the product decomposition and the supplied ingredient list
disagree, that is a coding error in the drug master, reported as
`coding_conflict` rather than silently resolved to one side.

This one change is what makes the model genuinely neutral: an HMIS with only
substance codes works, one with product codes works better, and one with both
gets its own data validated.

### 2.3 `local_id` is required

The HMIS's own drug master row identifier must be sent and is echoed in every
part of the response. It is the only thing that makes a coding error
*actionable*: "the finding you are questioning came from `DM-110877`, which you
have coded to Amoxicillin" points at one row in one table. Without it, debugging
a suspicious alert means guessing.

It carries no patient information and does not change the no-PHI posture.

## 3. Release skew — the HMIS's codes will go stale too

Previously, terminology drift meant *our* mapped SCTIDs going inactive
([C5](11-challenges-to-the-brief.md#c5)). Now the HMIS holds SCTIDs as well,
coded against whatever SNOMED release was current when the drug master was
coded — and drug masters are re-coded rarely. Within a few years the HMIS will
be sending codes our KB's release has inactivated.

`unknown_code` is an unhelpful answer to that. SNOMED already publishes the
answer, in the historical association reference sets (`SAME AS`, `REPLACED BY`,
`POSSIBLY EQUIVALENT TO`).

**Addition to the artifact:** an `IDX_HIST` section mapping inactivated SCTIDs to
their historical associations, restricted to concepts relevant to our mapped set.
Small — a few thousand entries — and it converts a dead end into a fix:

```jsonc
{ "ref": "1", "local_id": "DM-104422",
  "status": "stale_code",
  "reason": "inactive_in_kb_release",
  "message": "SCTID 12345678 was inactivated in the 20250401 release.",
  "replacement": { "code": "395821009", "display": "Pantoprazole",
                   "association": "SAME_AS" },
  "resolved_using_replacement": true }
```

Policy: for `SAME AS`, follow the replacement automatically and say so loudly in
the response. For `REPLACED BY`, follow it and mark the finding as depending on a
substitution. For `POSSIBLY EQUIVALENT TO`, **do not follow it** — report the
candidate and resolve nothing. The distinction matters: `SAME AS` is an assertion
of identity, `POSSIBLY EQUIVALENT TO` is an editorial hint, and treating the
second as the first is how an interaction gets attributed to the wrong molecule.

This also gives the HMIS team a re-coding worklist for free: every `stale_code`
with a replacement is a drug master row to update.

## 4. Deliverables that replace the adapter

Four, and together they cost less than the adapter would have.

### 4.1 Coding target set

`ddictl export --coding-set` produces the list of codes the KB actually
understands:

| Column | Notes |
|---|---|
| `sctid` | The substance or product code |
| `level` | `moiety` \| `salt` \| `product` |
| `preferred_name` | **Our** curated name (not the FSN — see [08 §3.2](08-licensing.md#32-consequence-for-kbddi)) |
| `ingredient_public_id` | What it resolves to |
| `covered` | Whether interaction data exists for it |
| `atc5`, `unii` | For cross-checking against the site's existing master |

The HMIS team codes against **this set**, not against the whole of SNOMED CT.
That is a far smaller and better-defined target, and it makes the coding exercise
tractable for a pharmacy team rather than requiring a terminologist per row.

Shipped in both licensing profiles; the `codes-only` profile is publishable to
non-affiliates, which makes this the artifact other Indian institutions can reuse
directly.

### 4.2 `/v1/resolve` as a pre-flight tool

Already in the API ([06 §2](06-api-contracts.md#2-other-rest-endpoints)), now
promoted to a headline deliverable. The HMIS team submits its entire coded drug
master and gets back, per row, what the service would resolve it to. No
interaction lookup, no patient context, runs offline, takes seconds.

This is the difference between discovering a coding error in production and
discovering it before go-live.

### 4.3 Drug master coding QA report

`ddictl qa-coding <drug-master.csv>` — the batch form of the above, with the
checks that only make sense in aggregate:

| Check | What it catches |
|---|---|
| Unresolved rate, by therapeutic area | Whole areas left uncoded |
| Rows coded to a **salt** where the moiety was intended | The commonest coding error |
| Rows whose code pulls > 3,000 interaction pairs | Coded to a class or grouper concept, not a substance |
| FDC rows with fewer codes than the composition string implies | A dropped component — [R5](09-risks.md), the highest-frequency safety gap |
| **Combi-pack rows coded to a single concept** | CDCI excludes combi packs, so a single code on an H. pylori / TB / peri-operative kit is necessarily wrong. The row must be decomposed into component products |
| Two drug master rows with identical composition but different codes | Internal inconsistency |
| Codes not in the coding target set | Coded to something we cannot use |
| Codes inactive in the KB's release | The re-coding worklist from §3 |

This is the quality gate that the previous draft got for free by controlling the
integration and now has to provide explicitly. It is the most important item in
this document.

### 4.4 Conformance kit

Published alongside the contract so any HMIS vendor can self-certify without us:

- OpenAPI document and CDS Hooks discovery document.
- A fixture set: ~40 request/response pairs covering every outcome-taxonomy
  branch, including all the awkward ones — FDC, salt-coded input, stale code with
  a `SAME AS` replacement, `POSSIBLY EQUIVALENT TO` (must *not* resolve),
  `coding_conflict`, unknown code, uncovered ingredient, route-gated
  non-finding, overlay suppression.
- `ddictl conformance --endpoint <url>` — runs the fixtures against a candidate
  integration and reports pass/fail per case.

Two of those fixtures matter more than the rest and should be mandatory to pass:
the integration must demonstrate that it **displays** the unresolved array, and
that it does **not** render `partial` as "no interactions found". Those are the
failures that harm patients, and they are integration-side failures we otherwise
have no way to detect.

## 5. The drug master coding exercise

The work that moved across the boundary. It is the institution's, but it will
fail without support, and its failure mode is silent wrong answers — so the plan
should treat it as a tracked dependency rather than someone else's problem.

**Size.** Unknown until the drug master is seen ([Q24](10-open-questions.md#q24)); a
tertiary hospital formulary is plausibly 3,000–8,000 rows. That is far more rows
than the 650 molecules in [01](01-phases.md) — but almost all of it is
mechanical, because the drug master already records composition, and the molecule
layer we curate is the target. The hard rows are those where composition parsing
fails, which is the same FDC and orthography problem the ranking tool already
solves.

**Reuse.** Point the existing machinery at a different input:

| Existing component | Reused for |
|---|---|
| N1–N7 normalizer ([03 §2](03-candidate-ranking.md#2-stage-1--normalization-deterministic-no-scoring)) | Parsing drug master composition strings |
| Candidate ranker | Proposing SCTIDs per row |
| Maker-checker workflow | Governing the coding, so it is auditable rather than a spreadsheet |
| `product` / `product_ingredient` tables ([02 §6](02-data-model.md#6-salt-and-product)) | Holding the result |

So the marginal build cost is small — this is mostly a second front-end view over
the same platform. The clinical/pharmacy cost is real and new.

**Estimate.** Auto-proposal should clear the large majority of rows. Assume 85%
auto-proposed and confirmed in bulk at ~15 s/row, 15% adjudicated at ~2 min/row:
5,000 rows ⇒ ~18 h bulk + ~25 h adjudication ≈ **43 h ≈ 1.5 person-weeks of
pharmacy time**, plus pharmacist review of the exceptions. Pharmacy technician
work with pharmacist sign-off, not consultant time.

**Governance.** This should go through maker-checker like everything else. A
mis-coded drug master row produces confidently wrong interaction findings for
every order of that product, indefinitely, and nobody downstream can see the
error. It deserves the same control as the molecule mappings — arguably more,
because it is higher-volume and lower-attention work.

## 6. Contract versioning

Neutrality only holds if the contract is stable enough to code against.

| Rule | |
|---|---|
| Path-versioned (`/v1/`); breaking changes take a new major path | |
| Additive changes (new optional request fields, new response fields, new enum members) are **not** breaking; integrators must ignore unknown fields, and the conformance kit tests that they do | |
| New enum members in `status` / `pair_status` / `outcome` are possible; integrators must have a default branch, and a fixture tests it | |
| Deprecation: 12 months' notice, both versions served in parallel | |
| The KB version and the contract version are **independent**. A KB release never changes the contract | |

That last row is the one integrators care about most: they should be able to take
KB updates — which is where all the clinical value lands — without touching code.

## 7. Risk shift

| Risk | Before | After | Why |
|---|---|---|---|
| [R19](09-risks.md) HMIS integration harder than assumed | 16 | **6** | No adapter, no protocol discovery, no dependency on a vendor change request |
| **R27 (new)** HMIS drug master mis-coded; service returns confidently wrong findings | — | **15** | Mitigated by the QA report, `/v1/resolve` pre-flight, maker-checker on coding, and `local_id` traceability |
| **R28 (new)** HMIS renders `partial` as "no interactions found", or drops the unresolved array | — | **12** | Mandatory conformance fixtures; the taxonomy is only as good as its rendering, and rendering is now entirely outside our control |
| **R29 (new)** HMIS codes go stale against newer KB releases | — | 8 | `IDX_HIST` historical associations, re-coding worklist |
| [R5](09-risks.md) FDC decomposition gaps | 16 | 16 | **Unchanged.** Moved from our ETL into the HMIS coding exercise — the QA report's FDC arity check is now the primary defence |

R28 deserves emphasis. The outcome taxonomy in
[06 §3](06-api-contracts.md#3-outcome-taxonomy) is the plan's main safety
mechanism, and neutrality hands its final presentation to integrators we do not
control. Publishing the distinction is no longer sufficient; it has to be
*tested* at the boundary, which is why two conformance fixtures are mandatory
rather than advisory.

## 8. Effort delta

| Change | Eng-weeks |
|---|---|
| HMIS adapter / shim | **−2.0** (and removes a 0–6 range) |
| Conformance kit (fixtures, `ddictl conformance`) | +1.0 |
| Coding target set export + `ddictl qa-coding` batch report | +1.0 |
| Drug master coding view in the curation UI | +1.0 |
| `IDX_HIST` historical associations (ETL + artifact + resolver) | +0.5 |
| Request model changes (`ingredients` vs `coding`, `local_id`, route/dose form) | +0.5 |
| **Net engineering** | **+2.0** |

| | Person-weeks |
|---|---|
| Engineering: 44.5 → **46.5** | |
| **Pessimistic case: 60 → ~53** | the adapter was the largest single unknown |
| Pharmacy (new): drug master coding | **+1.5–2.5** |

The planning number goes up slightly and the **variance drops sharply**, which is
the better trade. The residual unknown is the drug master itself, which can be
inspected in Phase 0 rather than discovered in Phase 5.

## 9. Consequent changes elsewhere

| Change | Where |
|---|---|
| Phase 5 reshaped: no adapter, no HMIS test instance dependency; conformance kit and coding support instead | [01](01-phases.md) |
| Request model: `local_id`, `ingredients[]` vs `coding[]`, `route`, `dose_form` | [06](06-api-contracts.md) |
| New statuses `coding_conflict`, `stale_code` with `replacement` | [06](06-api-contracts.md) |
| `IDX_HIST` artifact section; historical association policy | [04](04-etl-pipeline.md), [05](05-go-service.md) |
| `ddictl` gains `export --coding-set`, `qa-coding`, `conformance` | [05](05-go-service.md) |
| [C9](11-challenges-to-the-brief.md#c9) strengthened: REST-first was right, and is now the only integration surface we own | [11](11-challenges-to-the-brief.md) |
| [Q1](10-open-questions.md#q1) largely closed; replaced by [Q24](10-open-questions.md#q24)–[Q26](10-open-questions.md#q26) | [10](10-open-questions.md) |
| R19 down; R27–R29 added | [09](09-risks.md) |
