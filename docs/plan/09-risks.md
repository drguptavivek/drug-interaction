# 09 — Risk Register

Scored before mitigation. L = likelihood (1–5), I = impact (1–5).

Impact is judged on patient safety first, then on programme viability. A risk
that produces a wrong alert scores higher than one that delays the schedule by a
month, even though the latter is more visible to a steering committee.

## Clinical and knowledge risks

| ID | Risk | L | I | Score | Mitigation | Early warning |
|---|---|---|---|---|---|---|
| **R1** | **DDInter coverage of the Indian formulary is poor.** Molecules common in India (and Indian-specific FDCs) are absent, so the service silently evaluates nothing for them. | 4 | 5 | **20** | Phase 0 coverage census *before* committing. Explicit `ingredient_coverage` and a `not_evaluated` outcome, so absence is never reported as safety. Stop condition at < 70% NTI/QT coverage. Fallback: re-scope to a curated high-priority-list service. | Census results; `ddi_unresolved_total` and `ddi_pairs_not_evaluated_total` in production |
| **R2** | **Alert fatigue.** Interruptive tier too broad; clinicians click through everything, including the alerts that matter. | 4 | 5 | **20** | Shadow mode before go-live. ≤ 2 interruptive/100 orders as a gate. Structured `overrideReasons` as the feedback signal. Override rate > 80% is a documented stop condition that halts rollout. | Shadow-mode volumes; override rate in pilot |
| **R3** | **Prodrug/ester wrongly collapsed to parent moiety**, producing false alerts or, worse, false reassurance. | 3 | 5 | **15** | `derivation_kind` classification; ester/prodrug default to `no_collapse`; every collapse through maker-checker; band-A blocked on `unknown` ([C4](11-challenges-to-the-brief.md#c4)). | Batch consistency check: salts with differing `Is modification of` parents |
| **R4** | **Route-inappropriate alerts** (topical, ophthalmic, inhaled) drive clinicians to disable the system. | 4 | 3 | 12 | Route as a first-class product attribute; applicability predicate defaulting to systemic × systemic; `EXC_LOCAL_ROUTE` seeded ([C11](11-challenges-to-the-brief.md#c11)). | Departmental suppression requests clustering on topical products |
| **R5** | **FDC decomposition incomplete** — a component silently dropped, so its interactions are never checked. India is FDC-heavy, so this is a high-frequency failure. | 4 | 4 | **16** | Arity check (parsed component count vs resolved count); mismatch routed to human review; `resolved_via_product` response field lists every moiety so the caller can see the decomposition. | Arity mismatch rate during Phase 3 |
| **R6** | **Mapping error survives maker-checker** — a resident proposes and a rushed consultant approves. | 3 | 5 | **15** | Checker screen hides the tool ranking by default (anti-anchoring); interaction-count delta shown in clinical language; 10% blind re-audit with a ≥ 99% precision gate; batch consistency checks before release. | Re-audit precision; checker time-per-review trending down |
| **R7** | **Validation set is never collected**, because departments supply the top-50 list and stop. The project then has no external measure of recall. | 4 | 4 | **16** | Collect all three items in the *same* session; make the validation set a Phase 3 exit criterion; run it in CI so it cannot be quietly dropped. | Phase 3 tracking by department |
| **R8** | **Clinicians trust `no_interactions_found` when the real answer is `partial`.** | 3 | 5 | **15** | Outcome taxonomy separates them at the pair level; `no_interactions_found` is impossible when any pair is unevaluated (contract-tested); CDS Hooks emits an explicit info card for unevaluated drugs; training material addresses it directly. | Pilot interviews; override-reason analysis |

## Terminology and data risks

| ID | Risk | L | I | Score | Mitigation | Early warning |
|---|---|---|---|---|---|---|
| **R9** | **SNOMED concept inactivation** breaks mappings after a release upgrade. | 3 | 3 | 9 | `ddictl verify` fails the release; runtime degrades with `stale_code` rather than refusing to boot ([C5](11-challenges-to-the-brief.md#c5)); quarterly re-census. | Startup validation `ERROR` lines; `/healthz` degraded |
| **R10** | **CDC-India codes are not available in bulk**, or lack structured composition. | 3 | 4 | 12 | Phase 0 verification ([Q4](10-open-questions.md#q4)); fallback to the institutional formulary master as the product source, with CDC-India codes attached opportunistically. Product layer is deliberately decoupled from the moiety hub so this substitution is local. | Phase 0 |
| **R11** | **DDInter does not contain drug–food / drug–disease / duplication content**, contradicting the brief's scope assumption. | 4 | 3 | 12 | Verify in Phase 0 ([C3](11-challenges-to-the-brief.md#c3)); descope explicitly with stakeholder sign-off rather than quietly; structural moiety-duplication detection is still available at near-zero cost and should be labelled as structurally derived. | Phase 0 |
| **R12** | **Anchor disagreement rate is high**, making the third-anchor check noise rather than signal. | 3 | 2 | 6 | Measure in Phase 1 and report; normalise both sides to moiety before comparing UNII; ATC by set overlap ([C10](11-challenges-to-the-brief.md#c10)); retune thresholds before Phase 3. | Phase 1 anchor report |
| **R13** | **Indian drug names do not match SNOMED synonyms** (British spellings, local INN variants), collapsing band-A share and blowing up the curation estimate. | 3 | 3 | 9 | N6 orthographic normalization table, itself reviewed; measured band mix on the Phase 2 pilot with the estimate revised *before* Phase 3 commits clinician time. | Phase 2 pilot band mix |

## Legal and compliance risks

| ID | Risk | L | I | Score | Mitigation | Early warning |
|---|---|---|---|---|---|---|
| **R14** | **ShareAlike obligation attaches to the distributed binary** because the KB was embedded. | 4 | 4 | **16** | Sidecar artifact, separate licences, NOTICE in the tarball and in the artifact `META` ([C1](11-challenges-to-the-brief.md#c1)). Decision taken in Phase 0, not at packaging time. | — (design-time) |
| **R15** | **CredibleMeds content redistributed without permission.** | 3 | 4 | 12 | Not shipped; site-supplied overlay; `redistributable = false` enforced as a build gate ([C2](11-challenges-to-the-brief.md#c2)). | Build-time licensing check |
| **R16** | **SNOMED descriptions published to non-affiliates.** | 3 | 3 | 9 | `--profile codes-only` with a verify assertion; curated `preferred_name` distinct from FSN by design. | Verify stage |
| **R17** | **NC term challenged** for a fee-levying private ward, or a private hospital asks to deploy. | 2 | 3 | 6 | Counsel opinion in Phase 0; deployment guide states the boundary plainly. | — |
| **R18** | **Classified as regulated medical device software** under the Medical Devices Rules 2017. | 2 | 4 | 8 | Raise with institutional legal in Phase 0; position the service as advisory, clinician-in-the-loop, non-diagnostic; maintain an ISO 14971-style hazard log and clinical safety case from Phase 6 so that a later submission is a packaging exercise rather than a rebuild. | Legal review |

## Engineering and delivery risks

| ID | Risk | L | I | Score | Mitigation | Early warning |
|---|---|---|---|---|---|---|
| **R19** | **HMIS integration is far harder than assumed** — no FHIR, no CDS Hooks, a proprietary or database-level interface. | 4 | 4 | **16** | Confirm the actual protocol in Phase 3, not Phase 5 ([Q1](10-open-questions.md#q1)); REST-first sequencing; budget an adapter; escalate early if a test instance is not available. | Absence of a named HMIS contact by end of Phase 3 |
| **R20** | **Clinician availability collapses** and Phase 3 stalls. | 4 | 3 | 12 | Rotating resident makers; asynchronous work queue; department-count sequencing means the highest-value molecules are done first, so a truncated Phase 3 still yields a usable KB. Calendar buffer, not effort buffer. | Weekly adjudication throughput |
| **R21** | **Ranking tool underperforms**, band-A share far below 65%, curation effort doubles. | 3 | 3 | 9 | Calibrate on the Phase 2 50-molecule pilot and revise the estimate before Phase 3; terminologist allocated to the hard queue. | Phase 2 pilot |
| **R22** | **Artifact corruption in the field** (USB copy, failing disk at a PHC). | 3 | 3 | 9 | Per-section CRC; Ed25519 signature verified before interpretation, non-overridable; fuzz-tested reader that cannot panic on corrupt input. | Startup failures in the field |
| **R23** | **Overlay misconfiguration suppresses clinically important alerts** at a site. | 3 | 4 | 12 | Suppressions require rationale + approver; suppressed findings are *counted* in the response summary even when withheld; `ddi_overlay_suppressed_total` metric; overlay reviewed as a clinical artifact, not a config file. | Suppression counts trending up |
| **R24** | **Reproducible build broken**, so a released artifact cannot be regenerated when an error is reported months later. | 2 | 4 | 8 | Determinism asserted in CI; two-builder check at Phase 4 exit; full manifest in the tarball. | CI |
| **R25** | **Keycloak unavailable at a disconnected site**, blocking clinical use. | 3 | 4 | 12 | Offline JWT validation with a disk JWKS cache; `mode: static`; `mode: none` on loopback only ([C8](11-challenges-to-the-brief.md#c8)). Expiry of an auth cache must never deny care. | PHC pilot |
| **R26** | **No maintainer after handover.** The KB ages, the formulary moves, nobody notices. | 4 | 4 | **16** | Named institutional owner as a Phase 7 exit criterion; documented quarterly re-census (~0.5 person-week); `ddi_unresolved_total` as the automated canary; source-checksum change as a release-blocking event ([C6](11-challenges-to-the-brief.md#c6)). | Any quarter with no re-census |

## Top five by score

1. **R1** — DDInter coverage of the Indian formulary (20)
2. **R2** — Alert fatigue (20)
3. **R5** — FDC decomposition gaps (16)
4. **R7** — Validation set never collected (16)
5. **R14 / R16 / R19 / R26** — ShareAlike on the binary, HMIS integration, no maintainer (16)

R1 and R2 are the two that decide whether this system is used or quietly
switched off. Both are addressed by gates, not by good intentions: R1 by the
Phase 0 census and its stop condition, R2 by the shadow-mode alert-rate gate and
the pilot override-rate stop condition. If the programme is under schedule
pressure, those are the two gates that must not be waived.
