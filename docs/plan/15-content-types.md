# 15 — The Four Content Types

DDInter 2.0 supplies four kinds of interaction knowledge, not one. My earlier
draft planned for the first only, on a mistaken assumption
([C3, withdrawn](11-challenges-to-the-brief.md#c3)). This
document covers what each needs.

| Content | Records | Entities | Supporting text | Phase |
|---|---:|---|---|---|
| **DDI** drug–drug | 302,516 | 2,310 drugs | 8,398 distinct mechanism + management descriptions | 1 |
| **Duplication** therapeutic | 6,033 | 317 combination drugs, 96 pharmacological classes | warning + note per record | 1 |
| **DFI** drug–food | 857 | 29 foods | 430 mechanism + management descriptions | 2 |
| **DDSI** drug–disease | 8,359 | 472 diseases | 3,300 detailed records | 3 |

Sequencing is by *marginal cost*, not by record count — see §6.

---

## 1. The severity scale needs no change

DDInter grades on DRUGDEX criteria: **Major** (life-threatening and/or requiring
intervention), **Moderate** (may exacerbate disease and/or change therapy),
**Minor** (limits clinical effects, usually no therapy change), **Unknown**
(records from a *Sci Transl Med* dataset lacking mechanism descriptions).

That matches `ddi_severity` in [02 §10](02-data-model.md#10-interaction-rules-and-alert-tiering)
exactly. The `unknown` member is load-bearing: it is a real provenance class with
a known origin, not a null. Those records should default to **passive** tier and
never be interruptive — a rule with no mechanism description cannot be explained
to a clinician who overrides it.

## 2. The mechanism taxonomy is the most valuable part

More useful than the extra content types, and I would not have anticipated it.
DDInter annotates every interaction with a mechanism **category**:

| Category | Sub-mechanisms named | Route-sensitive? |
|---|---|---|
| **Absorption** | cation chelation; gastric-pH-dependent dissolution; intestinal P-gp inhibition/induction | **Yes — strongly** |
| **Distribution** | plasma protein binding competition; tissue transporter effects | No |
| **Metabolism** | CYP450 substrate/inhibitor/inducer (~40% of clinical DDIs) | No |
| **Excretion** | renal blood flow; tubular secretion competition; urinary pH | No |
| **Synergy** | additive/supra-additive effect | No |
| **Antagonism** | opposing effect | No |
| **Others** | description too ambiguous to categorise | Unknown |
| **Unknown** | no mechanism description available | Unknown |

### 2.1 This largely derives the route gate

[C11](11-challenges-to-the-brief.md#c11) argued that route must gate
applicability, and I planned it as hand-curated exception rules. The mechanism
category supplies a **defensible default** instead:

> `mechanism_category = 'absorption'` → the interaction is luminal. Default the
> applicability predicate to **enteral × enteral**; it does not apply to a
> parenteral route.

DDInter's own examples are exactly the cases I hand-listed:

| DDInter example | Category | What the default gives us |
|---|---|---|
| Antacid/PPI raising gastric pH, impairing itraconazole absorption | Absorption | Oral only — correct |
| Macrolide inhibiting intestinal P-gp, raising digoxin | Absorption | Oral only — correct |
| Ciprofloxacin/tetracycline chelated by dietary calcium | Absorption (DFI) | Oral only — correct |
| Iron chelating levothyroxine | Absorption | **Exactly the `EXC_IRON_ROUTE` exception I wrote by hand** |
| Ketoconazole (CYP3A4 inhibitor) × lovastatin | Metabolism | Systemic — no route gate, correct |

So `EXC_IRON_ROUTE` stops being a special case and becomes an instance of a rule
covering every absorption-mechanism interaction at once. That is a material
reduction in the hand-curated exception work estimated in
[07](07-effort.md#2-clinical-curation) — perhaps half of the ~40 exception rules.

**It is a default, not a conclusion.** Two guards:
- Topical and ophthalmic cases still need review. Ophthalmic timolol has systemic
  effect via a *synergy/antagonism* mechanism, so it is untouched by the
  absorption default — which is the right outcome, and worth verifying rather
  than assuming.
- `Others` and `Unknown` categories get **no** derived gate. They go to the hard
  queue.

### 2.2 It also sharpens the interruptive tier

Mechanism category is a better tiering input than severity alone. `Metabolism`
(CYP-mediated) and `Synergy` interactions are where the classic
life-threatening combinations sit — azole × statin (rhabdomyolysis),
benzodiazepine × opioid (respiratory depression), aminoglycoside × furosemide
(ototoxicity), MAOI × tyramine (hypertensive crisis). `Absorption` interactions
are usually manageable by separating doses — genuinely important, but *passive
with advice* rather than interruptive.

Proposed default tier, to be cut against shadow data in Phase 6:

| | Major | Moderate | Minor / Unknown |
|---|---|---|---|
| Metabolism, Synergy | **interruptive** | passive | passive |
| Absorption, Excretion, Distribution, Antagonism | passive + advice | passive | passive |
| Others, Unknown | passive | passive | passive |

## 3. Therapeutic duplication

**Do this in Phase 1, with DDI.** 6,033 records over 317 combination drugs and 96
pharmacological classes, each with a warning and note.

This closes a gap I had flagged: my earlier plan could only detect *exact moiety*
duplication structurally, which misses the clinically important case — two
different drugs of the same class. The named examples are the realistic ones:
two products both containing diphenhydramine, or two sedatives from different
prescribers.

| Needs | Notes |
|---|---|
| A `pharmacological_class` table and drug→class membership | 96 classes; DDInter supplies the membership |
| A duplication rule type keyed on **class**, not on a drug pair | The pair index is unordered-pair-keyed; class duplication is a different lookup — a per-request group-by over resolved ingredients' classes |
| A `duplication` finding type in the API | Distinct from `interaction`; see §5 |

Cheap, high clinical value, and India-relevant: FDC-heavy prescribing plus
multiple prescribers is precisely the duplication scenario.

**Do not conflate with the ATC hierarchy.** DDInter's 96 pharmacological classes
are its own; mapping them to ATC level 3/4 is tempting and would be a mapping
exercise with its own error modes. Use DDInter's classes as given, and map to ATC
only if a specific need appears.

## 4. Drug–food and drug–disease

### 4.1 DFI — Phase 2, small and worth it

857 records over **29 foods**. Twenty-nine is small enough to enumerate, curate
and code by hand in a day.

| Needs | Notes |
|---|---|
| A `food` entity, coded to SNOMED substance/food concepts | 29 rows; a bounded, one-off exercise |
| Extension of the request model | Food is not a prescribed item. It arrives as a **patient-advice** output, not a request input — see §5 |
| No route gate complexity | Food is enteral by definition |

The clinical value is concentrated and well known: grapefruit juice (CYP3A4),
alcohol (sedation with benzodiazepines and opioids), tyramine-rich foods with
MAOIs, vitamin-K-rich foods with warfarin, dairy calcium with fluoroquinolones
and tetracyclines. These are exactly the counselling points a discharge
prescription should carry, and they are cheap to add.

### 4.2 DDSI — Phase 3, and materially harder

8,359 records over **472 diseases**. The content is valuable; the cost is not in
the records but in everything around them.

| Needs | Why it is hard |
|---|---|
| **A third spoke**: disease → SNOMED **disorder** hierarchy | 472 concepts, a separate curation exercise with its own maker-checker cycle |
| **Patient problem list as request input** | The API currently takes coarse `renal_band` / `hepatic_impairment` flags. DDSI needs actual coded conditions |
| **PHI posture reconsidered** | A coded problem list is more identifying than a medication list. [05 §7](05-go-service.md#7-configuration-and-overlay-loading)'s no-PHI stance holds for codes without identifiers, but it needs an explicit review, not an assumption |
| **HMIS integration widens** | The caller must now send diagnoses, which many Indian HMIS deployments hold unreliably or as free text |

The last is the real obstacle and it is outside our control
([13](13-hmis-neutral-integration.md)). A DDSI check fed by an incomplete problem
list produces false reassurance — the same failure the outcome taxonomy exists to
prevent, in a domain where "no problem list" is common.

**Recommendation:** defer to Phase 3, and start with the subset that needs no
problem list — **renal and hepatic impairment**, which the API already accepts as
coarse bands, and which the DDInter documentation names as the commonest DDSI
cases. That delivers most of the clinical value with none of the integration
burden. Full problem-list DDSI comes later, or not at all.

## 5. API consequences

Findings are no longer homogeneous. The response needs a discriminated type:

```jsonc
"findings": [
  { "type": "interaction",  "pair": ["1","2"], "severity": "major",
    "mechanism_category": "metabolism", ... },

  { "type": "duplication",  "refs": ["3","5"],
    "pharmacological_class": "Antihistamines",
    "warning": "…", "note": "…" },

  { "type": "food",         "ref": "2", "food": "Grapefruit juice",
    "severity": "major", "mechanism_category": "metabolism",
    "audience": "patient_advice" },

  { "type": "disease",      "ref": "4", "condition": "Chronic kidney disease",
    "condition_source": "context.renal_band", "severity": "major" }
]
```

Rules:

- `type` is required and closed-set. Integrators must have a default branch for
  unknown members — a conformance fixture tests it
  ([13 §6](13-hmis-neutral-integration.md#6-contract-versioning)).
- Adding a type is **additive, not breaking**, so DFI and DDSI can ship in later
  KB releases without a contract version bump.
- `food` findings carry `audience: patient_advice`. They should **never** be
  interruptive at order entry — a grapefruit warning that blocks signing is the
  textbook alert-fatigue own goal. They belong on the printed prescription and in
  discharge counselling.
- `disease` findings state `condition_source`, so a clinician can see whether the
  condition came from a coded problem list or from a coarse band.
- The outcome taxonomy extends per type: `no_interaction_data` must be answerable
  separately for each, since a drug may have DDI coverage and no DFI coverage.

## 6. Effort

Marginal cost on top of the DDI build, assuming the bulk download carries all
four ([Q31](10-open-questions.md#q31)):

| Content | Engineering | Clinical | Phase | Why |
|---|---:|---:|---|---|
| **Mechanism taxonomy** | +0.5 | **−1.5** | 1 | Ingestion is trivial; it *removes* hand-curated route exceptions |
| **Therapeutic duplication** | +2.0 | +0.5 | 1 | Class table, class-based lookup, new finding type |
| **DFI** | +1.5 | +0.5 | 2 | 29 foods to code; patient-advice output path |
| **DDSI (renal/hepatic subset)** | +2.0 | +1.0 | 3 | Reuses the existing coarse context bands |
| **DDSI (full problem list)** | +6.0 | +3.0 | later | Disease spoke, problem-list input, PHI review, HMIS widening |
| **Totals excluding full DDSI** | **+6.0** | **+0.5** | | |

Engineering **46.5 → 52.5** person-weeks for Phases 1–3. Clinical is close to
flat, because the mechanism taxonomy gives back roughly what duplication and DFI
cost.

Compare with the [07 §4](07-effort.md#4-what-is-not-in-the-estimate) exclusion,
which put drug–disease and drug–food at "+6–10 eng-weeks, +4 clinical-weeks,
**plus a source**". The source exists, which was the hard part.

## 7. What this does not change

The architecture absorbs all four content types without redesign, which is worth
stating because it is the main test of whether the earlier design work holds up:

- **Hub-and-spoke** is unchanged. DFI adds a food entity; DDSI adds a disease
  spoke. Both hang off the same `ingredient` hub.
- **Moiety-level attachment** is unchanged.
- **The artifact format** takes new sections (`DUP`, `FOOD`, `DISEASE`) under the
  existing section table — that is what a section table is for
  ([04 §6](04-etl-pipeline.md#6-artifact-format)).
- **CSR pair indexing** is unchanged for DDI. Duplication is a class group-by;
  food and disease are per-drug lookups, both cheaper than a pair probe.
- **String interning is now quantified and validated**: 302,516 DDI records share
  8,398 distinct mechanism/management descriptions — roughly **36× reuse**. The
  prediction in [04 §6](04-etl-pipeline.md#6-artifact-format) that interning would
  cut the prose section by an order of magnitude was, if anything, conservative.
- **Maker-checker, append-only curation, releases** — all unchanged.
