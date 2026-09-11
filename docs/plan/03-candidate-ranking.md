# 03 — Candidate Ranking for the Mapping Tool

The design principle from the brief is right and worth restating: **the human
adjudicates, the machine searches.** A consultant should never be typing into a
SNOMED free-text box. The tool's job is to put the correct answer in position 1
often enough that review is fast, and — more importantly — to make it *visible*
when it is not confident.

## 1. Input

One unit of work = one **departmental submission row**, e.g.:

```
"Tab. Amoxycillin 500mg + Clavulanic Acid 125mg BD"
"Inj. Pantoprazole 40 mg IV OD"
"Ferrous ascorbate 100mg + Folic acid 1.5mg"
"T. Dytor plus"                       <- brand; expect these
```

## 2. Stage 1 — Normalization (deterministic, no scoring)

| Step | Action | Example |
|---|---|---|
| N1 | Strip dose-form prefixes | `Tab.`, `Cap.`, `Inj.`, `Syp.`, `T.`, `C.` → captured as `dose_form` hint |
| N2 | Strip frequency/timing | `BD`, `TDS`, `OD`, `HS`, `SOS`, `Q6H` |
| N3 | Extract strength tokens | `500mg`, `1.5 mg`, `40 mg` → `strength` list |
| N4 | Extract route tokens | `IV`, `IM`, `PO`, `topical`, `eye drops` → `route_class` hint |
| N5 | Split FDC | on `+`, `/`, `and`, `with` → **n component strings** |
| N6 | Orthographic normalization | `Amoxycillin`→`amoxicillin`, `Sulph`→`sulf`, `Frusemide`→`furosemide`, `Paracetamol`↔`acetaminophen` |
| N7 | Brand detection | lookup against CDC-India/institutional brand list → resolve to composition, flag `via_brand` |

N6 matters more in India than most places: British spelling is standard in
Indian prescribing while DDInter and UNII use USAN/INN American forms. The
mapping table is small (a few hundred entries), hand-built once, and is itself a
reviewed artifact.

**Failure mode to design for:** N5 splits `"Amoxycillin + Clavulanic acid"`
correctly but also splits `"Trimethoprim and Sulfamethoxazole"` — fine — and
mis-splits `"Vitamin B complex with C"` — not fine. Any row where N5 produces
components that fail to resolve is routed to a human *as a whole row*, with the
split shown and editable. Never let a bad split silently become two bad mappings.

## 3. Stage 2 — Candidate generation (recall-oriented, ~20 candidates)

Eight independent retrievers. Independence is the point: corroboration across
retrievers is a scoring feature, so retrievers must not share a code path.

| ID | Retriever | Index | Typical yield |
|---|---|---|---|
| R1 | SNOMED exact — FSN and synonym exact match within the substance hierarchy | `citext` btree | 0–1, very high precision |
| R2 | SNOMED fuzzy — trigram similarity over FSN + all active synonyms | `pg_trgm` GIN | 5–10 |
| R3 | SNOMED via product — CDC-India/SNOMED product concept → `has active ingredient` → substance | relationship join | 0–3, high precision when brand resolved |
| R4 | DDInter name match — exact then trigram over DDInter drug names | `pg_trgm` | 0–5 |
| R5 | UNII/GSRS name match — preferred term and all synonyms, then UNII → SNOMED via the UNII map refset **if the release contains one** ([Q22](10-open-questions.md#q22)); otherwise via R8 | join | 0–2, very high precision |
| R6 | ATC name match — WHO ATC index name → ATC-5 → members | join | 0–3 |
| R7 | RxNorm **name** match at IN/PIN level — weak, and never a sole basis for a proposal | join | 0–2 |
| R8 | RxNorm **structural** — `DDInter → DrugBank → RXCUI → SNOMEDCT_US → SCTID`, and `SCTID → RXCUI → UNII`. No string matching at any step | join | 0–2, very high precision |

R7 and R8 are the same source and deserve opposite treatment — see
[12 §B5](12-terminology-tooling.md#b5-consequent-changes-to-the-ranking-algorithm).
R8 is a structural path and is strong evidence; R8 may also be the **only**
derivation of the SNOMED-side UNII anchor if the release ships no UNII map
refset. R7 is lexical and is weak. RxNorm **brand** names (`BN`, `SBD`) are
prohibited as evidence entirely: Indian and US brand names collide frequently
for different molecules, so a brand match is a false-evidence generator — worse
than no evidence, because it looks like corroboration.

R7's constraint is deliberate: RxNorm reflects the US market. An Indian molecule
absent from RxNorm is unremarkable, and a *present* RxNorm match on an Indian
brand name is frequently a coincidental homonym. The brief already says RxNorm is
for international interoperability; the ranker enforces that by capping its
contribution and refusing to preselect a candidate supported by R7 alone.

Candidates are then **normalised to moiety**: any candidate that is a salt (has
an outgoing `Is modification of` edge classified as `salt`) is replaced by its
parent, with the salt recorded. Candidates classified as `ester`, `prodrug` or
`complex` are **not** replaced — they are presented as distinct candidates with a
visible badge, because that is exactly the decision the clinician must make
([C4](11-challenges-to-the-brief.md#c4)).

## 4. Stage 3 — Scoring

A linear score over bounded features. Deliberately not a learned model:
there is no training data at project start, ~650 decisions is far too few to fit
one, and a clinician must be able to read why a candidate ranked first.

```
score(c) = Σ w_i · f_i(c)          clipped to [0, 100]
```

| # | Feature `f_i` | Range | `w_i` | Notes |
|---|---|---|---|---|
| F1 | Exact match on an active SNOMED FSN/synonym | 0/1 | **30** | |
| F2 | Lexical similarity (token-set Jaro-Winkler, post-N6) | 0–1 | 20 | |
| F3 | UNII agreement between the SNOMED side and the DDInter side | 0/1 | **25** | the strongest single signal |
| F4 | ATC-5 **set overlap** (Jaccard) | 0–1 | 10 | §5 |
| F5 | Retriever corroboration count, `min(k,4)/4` | 0–1 | 10 | independence matters |
| F6 | Candidate is a moiety (no outgoing `Is modification of`) | 0/1 | 8 | |
| F7 | Candidate is in the India extension or the Indian NLEM | 0/1 | 5 | Indian-market prior |
| F8 | Candidate already mapped by an approved proposal for a sibling submission | 0/1 | 5 | consistency across departments |
| F9 | Brand resolution path used (R3) and composition matched arity | 0/1 | 7 | |
| P1 | Candidate concept is inactive in the pinned release | 0/1 | **−40** | |
| P2 | Candidate is an ester/prodrug/complex, not a salt | 0/1 | −15 | forces a conscious decision |
| P3 | UNII **disagreement** between sides | 0/1 | **−35** | |
| P4 | Supported by R7 only | 0/1 | −20 | |
| F10 | RxNorm **structural** path corroborates the candidate (R8) | 0/1 | **20** | Often the same evidence as F3; independent when the UNII refset is absent |
| F11 | RxNorm IN/PIN agrees with the proposed `derivation_kind` | 0/1 | 8 | Independent second opinion on salt collapse |
| P5 | Homonym risk: candidate name is also a common non-drug English word, or matches ≥ 3 unrelated substances | 0/1 | −10 | |
| P6 | RxNorm treats as a distinct `IN` what SNOMED marks as a modification | 0/1 | **−20** | Strong prodrug/ester signal; blocks band A |

**Bands and what they do:**

| Band | Score | Tool behaviour |
|---|---|---|
| A | ≥ 75, **and** no blocking condition | Top candidate pre-selected in the maker's form. Maker still confirms. |
| B | 45–74 | Ranked list shown, nothing pre-selected. |
| C | < 45, or zero candidates | Routed to "hard queue", reviewed in a batch with a terminologist present. |

**Blocking conditions** (force band B or worse regardless of score):
UNII disagreement; candidate inactive; candidate not in the substance hierarchy;
FDC where component count ≠ resolved component count; `derivation_kind = 'unknown'`; RxNorm/SNOMED collapse disagreement (P6).

Band A never bypasses maker-checker. It changes one thing only: whether a radio
button starts selected. The brief's workflow is untouched.

**Weights are a starting point, not a result.** They are calibrated once, on the
Phase 2 50-molecule pilot, by measuring top-1 accuracy and the band-A error rate.
The target is band-A top-1 precision ≥ 98%; if the pilot shows less, weights are
retuned before Phase 3 and `proposal.tool_version` records which weight set
produced each proposal so the two cohorts remain distinguishable.

## 5. Anchor agreement semantics

| Anchor | Cardinality | Comparison | On disagreement |
|---|---|---|---|
| UNII | 1 per substance (salt and moiety have *different* UNIIs — always compare at the same level) | equality after moiety normalization | **Blocking.** Red banner, no preselect. |
| ATC-5 | **many** per moiety | non-empty set intersection | Advisory. Amber note showing both sets. |
| RxCUI | 1 per IN | equality | Informational only. |

The UNII row hides a trap worth stating: DDInter identifies drugs at a level that
is sometimes the salt and sometimes the moiety, so a naive UNII comparison
produces systematic false disagreements on every salt-listed drug. The comparison
must normalise both sides to moiety *first*, via the GSRS relationship, and
record in `anchor_observation.derivation` which path was used.

## 6. Evidence shown to the maker

One screen, no scrolling for the common case.

```
┌─ Submission ────────────────────────────────────────────────────────────┐
│ "Inj. Pantoprazole 40 mg IV OD"      Cardiology (rank 12) · 6 departments│
│ normalized: pantoprazole | route: systemic_parenteral | strength: 40 mg  │
└──────────────────────────────────────────────────────────────────────────┘

┌─ Candidate 1  score 92  BAND A ─────────────────────────── [ ● selected ]┐
│ SNOMED  395821009 | Pantoprazole (substance) |            ACTIVE  Intl   │
│ DDInter DDInter-00841  "Pantoprazole"                                    │
│ UNII    D8TST4O562  ✓ agrees (snomed→rxcui→unii ↔ ddinter→gsrs)          │
│ ATC-5   A02BC02  ✓ overlap 1/1                                           │
│ moiety  yes — no 'Is modification of' edges                              │
│ RxNorm  typed IN, not PIN  ✓ agrees this is the moiety, not a salt       │
│ found by R1 exact, R2 trg 0.97, R4 exact, R5 unii, R6 atc,               │
│          R8 structural (ddinter→drugbank→rxcui→sctid)   6/8              │
│ ─ salts that will collapse into this moiety ──────────────────────────── │
│   pantoprazole sodium (sctid 428204000)  derivation: salt                │
│   pantoprazole sodium sesquihydrate      derivation: salt                │
│ ─ what this will mean ────────────────────────────────────────────────── │
│   1,204 DDInter pairs attach  ·  3 major  ·  ONCHigh: no                 │
└──────────────────────────────────────────────────────────────────────────┘

┌─ Candidate 2  score 41 ──────────────────────────────────────────────────┐
│ SNOMED  429386002 | Pantoprazole sodium (substance) |     ACTIVE         │
│ ⚠ not a moiety — 'Is modification of' → 395821009                        │
└──────────────────────────────────────────────────────────────────────────┘

[ Propose candidate 1 ]  [ Propose other… ]  [ Flag exception ]  [ Cannot map ]
```

Identifiers in this mockup are illustrative, not verified values.

The "what this will mean" block is the part most mapping tools omit and the part
that catches the worst errors. A consultant who cannot tell two SCTIDs apart can
immediately tell that one of them pulls in 1,204 interaction pairs and the other
pulls in 3.

`Cannot map` is a first-class outcome, not a failure. It produces a proposal with
`payload.decision = 'no_mapping'` and a reason code, which is reviewed like any
other and which produces an explicit `ingredient_coverage.is_covered = false`
row — feeding straight into the `no_interaction_data` outcome at query time.

## 7. Evidence shown to the checker — different, on purpose

The checker is a consultant with less time and a different job: not "is this
plausible" but "would I sign this". The checker's screen shows everything above
**plus** the following, and hides the ranked list by default so the checker
is not anchored by the tool's opinion before forming their own:

| Additional element | Why |
|---|---|
| Who proposed it, when, how long they spent | Fatigue and rubber-stamping are visible |
| Whether the maker took the pre-selected band-A candidate or overrode it | An override is a signal worth reading |
| The **runner-up** candidate and the score gap | A gap of 92 vs 88 is a different review than 92 vs 41 |
| Diff against any superseded proposal for the same `entity_key` | Corrections get scrutiny |
| Every anchor disagreement, expanded, not collapsed | Cannot be skipped |
| Interaction-count delta, and the **named** major interactions | The clinical consequence, in clinical language |
| Other departments that submitted this molecule | Cross-specialty context |
| `[ Show tool ranking ]` — collapsed by default | Reduces automation bias |

Checker actions: `Approve`, `Reject with reason`, `Request changes`. A rejection
must carry a reason code; a free-text-only rejection is refused by the API,
because "rejected: wrong" teaches the maker nothing and teaches the weight
calibration nothing.

## 8. Batch consistency checks (run before release assembly, not per-proposal)

These catch the errors that are invisible one row at a time:

| Check | Failure means |
|---|---|
| Two ingredients share a UNII | A missed merge |
| Two ingredients share an ATC-5 **and** high name similarity | Probable duplicate |
| An ingredient has salts whose `Is modification of` parents differ | Collapse error |
| A DDInter ID appears on two ingredients without an approved `spoke_ddinter_secondary` | Spoke violation |
| An ingredient with `is_covered = true` has zero attached rules | Likely a spoke mis-mapping |
| An ingredient with > 3,000 attached pairs | Likely mapped to a class concept, not a substance |
| A validation-set pair that the built KB does not find | A real recall failure, investigated individually |

The last one is the one that matters: it is the only check in this document that
tests the system against clinicians' actual knowledge rather than against itself.

## 9. Throughput assumptions

Calibrated in Phase 2; these are the planning figures:

| Band | Share of rows | Maker | Checker |
|---|---|---|---|
| A | 65% | 2 min | 1 min |
| B | 25% | 6 min | 3 min |
| C | 10% | 20 min | 8 min |

650 molecules ⇒ maker ≈ 650 × (0.65·2 + 0.25·6 + 0.10·20) min ≈ 52 h;
checker ≈ 25 h. Plus FDC decomposition and exception adjudication — see
[07](07-effort.md).
