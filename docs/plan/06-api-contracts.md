# 06 — API Contracts

Two surfaces over one engine. Sketches, not final specs — the OpenAPI document
is a Phase 4 deliverable.

## 1. `POST /v1/interactions/check`

For internal applications. Synchronous, stateless, no patient identifiers.

### Request

```jsonc
{
  "request_id": "b3f1…",                 // caller-supplied, echoed, for log correlation
  "drugs": [
    { "ref": "1", "coding": [{ "system": "http://snomed.info/sct", "code": "372756006" }],
      "route": "systemic_oral" },
    { "ref": "2", "coding": [{ "system": "urn:cdc-india", "code": "CDC-IN-10422" }] },
    { "ref": "3", "coding": [{ "system": "urn:ddinter", "code": "DDInter-00841" }] },
    { "ref": "4", "text": "Tab. Amoxycillin 500 + Clav 125" }   // accepted, never trusted
  ],
  "context": {                            // all optional, all coarse
    "age_band": "65_plus",
    "renal_band": "egfr_30_59",
    "hepatic_impairment": false,
    "pregnancy": false
  },
  "options": {
    "include_passive": true,
    "include_not_evaluated": true,
    "max_findings": 200
  }
}
```

Design notes:

- `ref` is caller-assigned and is how every output element points back at an
  input. Positional indexing breaks the moment a caller filters its own list.
- `coding` is an array: a caller may supply SCTID *and* a local product code, and
  agreement between them is extra evidence. Disagreement is reported, not
  silently resolved to the first one.
- `text` is accepted because real HMIS integrations will send it, and refusing it
  means integrators will fabricate a code instead. But a `text`-only drug can
  never be `resolved` — its best outcome is `resolved_low_confidence`, which the
  response marks and which the CDS Hooks surface refuses to alert on.
- `context` is accepted and, in phase 1, **only echoed**. Renal- and
  age-dependent rules are not in scope; accepting the fields now keeps the
  contract stable when they are. The response states `"context_applied": false`
  so no caller can believe otherwise.

### Response

```jsonc
{
  "request_id": "b3f1…",
  "kb": {
    "version": "ddinter2.0+snomedIN20250401+r7",
    "built_at": "2026-03-14T00:00:00Z",
    "status": "ok",                       // ok | degraded
    "degraded_reasons": [],
    "overlay_version": "aiims-2026.1"
  },
  "context_applied": false,
  "summary": {
    "outcome": "interactions_found",
    "drugs_submitted": 4,
    "drugs_resolved": 3,
    "pairs_possible": 6,
    "pairs_evaluated": 3,
    "pairs_not_evaluated": 3,
    "findings": { "interruptive": 1, "passive": 2, "suppressed": 1 }
  },

  "resolved": [
    { "ref": "1", "status": "resolved", "via": "direct_substance",
      "ingredients": [{ "id": "ING-000088", "name": "Levothyroxine",
                        "sctid": "372756006" }],
      "route": "systemic_oral", "covered": true }
  ],

  "unresolved": [                          // never empty-by-omission; always present
    { "ref": "4", "status": "resolved_low_confidence",
      "reason": "text_only_input",
      "message": "Input supplied as free text; no code provided. Not evaluated.",
      "suggestions": [ { "id": "ING-000210", "name": "Amoxicillin", "score": 0.81 } ] }
  ],

  "findings": [
    {
      "finding_id": "f1",
      "pair": ["1", "2"],
      "ingredients": ["ING-000088", "ING-000417"],
      "severity": "major",
      "tier": "interruptive",
      "onchigh": true,
      "mechanism": "Polyvalent cation chelation reduces levothyroxine absorption.",
      "management": "Separate administration by at least 4 hours; monitor TSH.",
      "attached_at": "moiety",
      "applicability": { "route_gate": "systemic_oral x systemic_oral", "fired": true },
      "provenance": [
        { "source": "DDINTER", "version": "2.0", "ref": "DDInter-00841_DDInter-01122" },
        { "source": "ONCHIGH", "version": "2016" }
      ]
    }
  ],

  "not_evaluated": [
    { "pair": ["1", "3"], "reason": "ingredient_not_covered",
      "uncovered": ["ING-000902"],
      "message": "No interaction data exists for Serratiopeptidase. This is NOT a finding of no interaction." }
  ]
}
```

The `message` on the last element is the entire point of the outcome taxonomy,
written out in the wire format so that a lazy integrator rendering
`not_evaluated[].message` verbatim still tells the clinician the truth.

## 2. Other REST endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/kb` | KB version, build metadata, source list, licence NOTICE |
| `GET` | `/v1/ingredients/{id}` | Moiety detail, salts, codes, coverage |
| `GET` | `/v1/ingredients/{id}/interactions` | Full neighbour list (uses the CSR row directly) |
| `POST` | `/v1/resolve` | Resolution only, no interaction lookup — for formulary mapping tools |
| `GET` | `/healthz` | `ok` \| `degraded`, with reasons |
| `GET` | `/readyz` | KB loaded and verified |
| `GET` | `/metrics` | Prometheus |

## 3. Outcome taxonomy

Three independent levels. Conflating them is the failure this taxonomy exists to
prevent.

### 3.1 Per-input resolution status

| Status | Meaning | Appears in |
|---|---|---|
| `resolved` | Code mapped to ≥ 1 active moiety | `resolved[]` |
| `resolved_via_salt` | Mapped through a salt concept; salt recorded | `resolved[]` |
| `resolved_via_product` | Product decomposed; may yield several moieties | `resolved[]` |
| `resolved_low_confidence` | Free text or fuzzy match; **not used for alerting** | `unresolved[]` |
| `ambiguous` | Code maps to > 1 moiety with no decomposition available | `unresolved[]` |
| `stale_code` | Code known to the KB but inactive in the local SNOMED release | `unresolved[]` |
| `unknown_code` | Code not in the KB | `unresolved[]` |
| `unsupported_system` | Coding system not understood | `unresolved[]` |
| `excluded_by_overlay` | Not on this institution's formulary | `unresolved[]` |

Anything not `resolved*` lands in `unresolved[]`. There is no path by which an
input disappears from the response: a contract test enumerates the response and
asserts every submitted `ref` appears exactly once across `resolved` and
`unresolved`.

### 3.2 Per-pair status

| Status | Meaning |
|---|---|
| `interaction` | Rule found and applicable |
| `evaluated_no_interaction` | Both ingredients covered; no rule. **This is a real negative.** |
| `not_evaluated_uncovered` | ≥ 1 ingredient absent from the source's drug list |
| `not_evaluated_unresolved` | ≥ 1 input did not resolve |
| `not_applicable_route` | Rule exists; route gate suppressed it |
| `not_applicable_exception` | Rule exists; a curated exception suppressed it |
| `suppressed_by_overlay` | Institutional suppression |

`not_applicable_*` and `suppressed_by_overlay` are reported (in
`not_evaluated[]`, with reason) rather than hidden. A clinician who overrides
needs to know a rule was considered and set aside, and a quality team needs to
audit whether the suppressions are right.

### 3.3 Request-level outcome

Computed, in this priority order:

| `outcome` | Condition |
|---|---|
| `interactions_found` | ≥ 1 finding survived tiering |
| `no_interactions_found` | **all** pairs `evaluated_no_interaction` |
| `no_interaction_data` | **no** pair could be evaluated |
| `partial` | Some pairs evaluated, some not, and no findings |
| `invalid_request` | Schema violation |

`partial` is the case that most APIs of this kind get wrong by rounding it to
"no interactions found". It is separated here precisely because it is the common
case in Indian practice, where a list of five drugs routinely contains one the
knowledge base has never heard of.

## 4. CDS Hooks surface

Per the HL7 PDDI-CDS implementation guide, with two hooks.

### 4.1 Discovery — `GET /cds-services`

```jsonc
{
  "services": [
    {
      "hook": "order-select",
      "id": "ddi-check-order-select",
      "title": "Drug–drug interaction check (India)",
      "description": "Screens a selected medication order against the DDI knowledge base.",
      "prefetch": {
        "medications": "MedicationRequest?patient={{context.patientId}}&status=active"
      }
    },
    {
      "hook": "order-sign",
      "id": "ddi-check-order-sign",
      "title": "Drug–drug interaction check at signature (India)",
      "prefetch": { "medications": "MedicationRequest?patient={{context.patientId}}&status=active" }
    }
  ]
}
```

### 4.2 Response cards

```jsonc
{
  "cards": [
    {
      "uuid": "…",
      "summary": "Major: Levothyroxine + oral iron — reduced absorption",
      "detail": "Polyvalent cation chelation reduces levothyroxine absorption.\n\n**Management:** separate administration by ≥ 4 hours; monitor TSH.",
      "indicator": "warning",
      "source": {
        "label": "Indian DDI Knowledge Service ddinter2.0+snomedIN20250401+r7",
        "url": "https://…/v1/kb"
      },
      "overrideReasons": [
        { "code": "separated-administration", "system": "urn:ddi:override",
          "display": "Administration times already separated" },
        { "code": "benefit-outweighs", "system": "urn:ddi:override",
          "display": "Benefit outweighs risk; will monitor" },
        { "code": "patient-tolerating", "system": "urn:ddi:override",
          "display": "Patient already tolerating this combination" },
        { "code": "not-clinically-significant", "system": "urn:ddi:override",
          "display": "Not clinically significant in this patient" }
      ],
      "links": [ { "label": "Evidence and provenance", "url": "…", "type": "absolute" } ]
    },
    {
      "uuid": "…",
      "summary": "1 drug could not be evaluated: Serratiopeptidase",
      "detail": "No interaction data exists for this drug. Absence of an alert does not mean absence of an interaction.",
      "indicator": "info",
      "source": { "label": "Indian DDI Knowledge Service …" }
    }
  ]
}
```

Rules for this surface:

| Rule | Reason |
|---|---|
| `indicator: critical` is reserved for the curated interruptive tier — **never** driven by DDInter severity alone | Severity is a property of the interaction; interruptiveness is an institutional decision |
| `order-select` emits `info`/`warning` only | Interrupting during selection is the fastest route to alert fatigue |
| `order-sign` may emit `critical` | Signature is the last safe moment and the one clinicians expect to be stopped at |
| Unevaluated drugs always produce an `info` card | The unresolved array must survive translation into the EMR's UI |
| `overrideReasons` are structured and mandatory on `critical` cards | Override reasons are the only data that tells you whether your tiering is wrong |
| Every card carries the KB version in `source.label` | A clinician asking "why did it say that" can be answered six months later |

### 4.3 What I would sequence differently

The brief treats CDS Hooks as the HMIS integration path. Most Indian
public-sector HMIS deployments — including much of the eHospital/NIC estate — are
not FHIR servers and cannot originate CDS Hooks calls with a `MedicationRequest`
prefetch. See [C9](11-challenges-to-the-brief.md#c9). Plan accordingly:

1. REST first. It is what the AIIMS HMIS will actually be able to call.
2. CDS Hooks second, built to the IG, for the deployments that can use it and for
   standards conformance — which has real value for ABDM alignment and for
   publication.
3. A thin adapter (`ddi-hmis-shim`) translating whatever the AIIMS HMIS speaks
   into the REST call. Budget for this in Phase 5 and confirm the actual protocol
   in Phase 3. If it turns out the HMIS can only do a synchronous database
   callout, that is a different and larger integration, and it is far better to
   discover it in month 4 than in month 6.

## 5. Cross-surface invariants (contract-tested)

1. The same medication list produces the same findings on both surfaces.
2. No input is ever absent from the response.
3. `no_interactions_found` is impossible if any pair is `not_evaluated_*`.
4. Every finding carries `kb.version` and at least one provenance entry.
5. A `resolved_low_confidence` input never contributes to a `critical` card.
6. Suppressed findings are counted in `summary.findings.suppressed` even though
   their content is not returned — the count alone is what lets a quality team
   notice an over-aggressive overlay.
