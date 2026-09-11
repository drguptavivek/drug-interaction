# CLAUDE.md — working context for this repository

## What this is

An implementation plan for a **drug–drug interaction (DDI) knowledge service for
Indian healthcare**: a Go API callable from HMIS/EMR systems, built for an
AIIMS-level tertiary hospital and deployable unchanged to district hospitals,
CHCs and PHCs with different formularies and no reliable connectivity.
Non-commercial; government and academic deployment only.

**Current state: plan only. No production code has been written yet.** The
deliverable so far is `README.md` plus `docs/plan/00`–`13`. Start there before
proposing anything.

## Repository map

```
README.md                 index + headline conclusions
docs/plan/00–13           the plan (see README table)
snomed-releases/          LOCAL ONLY — licensed release archives, gitignored
.gitignore                a compliance control; read the comment at the top
```

The three documents that carry the most weight:

| Doc | Why it matters |
|---|---|
| `docs/plan/11-challenges-to-the-brief.md` | Twelve constraints in the original brief I argued are mistakes, with reasons. Read before re-proposing any of them. |
| `docs/plan/08-licensing.md` | What may and may not be published. This repo is **public**. |
| `docs/plan/10-open-questions.md` | Q1–Q26. Several are Phase 0 gates. |

## Hard rules

These are settled decisions with reasoning recorded. Don't quietly reverse them;
if one looks wrong, argue it against the document that sets it out.

0. **India is a SNOMED Member country — the Affiliate Licence is free.** Nothing
   here is blocked by SNOMED cost. The obligations are *registration* (via
   NRCeS, gates Phase 2 — a non-Affiliate may not copy SCTIDs into a database)
   and an **annual Declaration of Use, due 15 January**, which needs an
   administrative deployment register. Do not solve the register with telemetry;
   that would breach rule 5 for a paperwork purpose. (`08 §3.6`)
1. **Never commit licensed source data.** The repo is public. SNOMED CT RF2,
   DDInter, RxNorm/UMLS and CredibleMeds content are each restricted in ways a
   public commit would breach. Sources are pinned by SHA-256 in `sources.lock`
   and fetched locally. (`08-licensing.md`, `snomed-releases/README.md`)
2. **The knowledge base is never embedded in the binary.** No `go:embed` of KB
   content — it attaches the CC BY-NC-SA ShareAlike obligation to the compiled
   artifact. `ddid` is Apache-2.0 with no knowledge content; `kb.ddi` ships as a
   signed sidecar under CC BY-NC-SA 4.0. (`11` C1, `08 §2`)
3. **CredibleMeds data is never shipped.** Sites supply it as an overlay under
   their own registration. (`11` C2)
4. **No terminology server at runtime.** CSNOServ / BHTS / Snowstorm are build-
   and curation-time only. `ddid` links no terminology client and has no config
   key for one — deliberately, so the offline property can't be eroded. CSNOtk
   being Apache-2.0 does **not** make it safe to link `CSNOLib` into `ddid`: the
   constraint is the offline property and the SNOMED content, not the code
   licence. (`12 §A1`, `§A3`, `05`)
5. **No network dependency at query time**, including auth. JWT is validated
   offline against a cached JWKS; an expired auth cache must never deny care.
   (`11` C8, `05 §8`)
6. **No PHI.** Requests carry coded medication lists and coarse context only.
   `audit.include_patient_id` is rejected if set. (`05 §7`)
7. **Maker-checker is enforced in the database**, not the UI and not the service
   layer — a trigger, so it holds against direct `psql` too. Proposals and
   reviews are append-only (`UPDATE`/`DELETE` revoked). (`02 §8`)
8. **`no_interaction_data` ≠ `no_interactions_found`**, at the *pair* level, not
   just the request level. `no_interactions_found` is impossible when any pair
   went unevaluated. This is the plan's main safety mechanism. (`06 §3`)
9. **Salt→moiety collapse is machine-proposed, human-approved.** Never fully
   automated from `Is modification of` — that relationship also covers esters,
   prodrugs and complexes. (`11` C4)
10. **Terminology drift fails the release in CI, never the running process.**
    A CDS service that won't boot removes interaction checking silently.
    (`11` C5, `05 §6`)
11. **HMIS-neutral.** No vendor- or site-specific code in scope. Site variation
    lives in the overlay file, which is data. (`13`)

## Local-only data

`snomed-releases/` holds SNOMED CT release archives on the developer's machine.
The folder is tracked (via `.gitkeep`) so the layout is discoverable; its
contents are ignored. See `snomed-releases/README.md` for expected structure and
for how to record a release in `sources.lock`.

Other licensed sources get sibling directories, already gitignored: `ddinter/`,
`rxnorm/`, `umls/`, `credible-meds/`, `cdc-india/`, `drugbank/`.

## Where the work picks up: Phase 0

**Follow [`docs/plan/14-phase-0-runbook.md`](docs/plan/14-phase-0-runbook.md)** —
ordered steps P1–P11 with owners, durations and the go/no-go gate. Summary of
what matters:

- [ ] **Coverage census** — 150 molecules from the AIIMS formulary (50
      high-volume, 50 random, 50 NTI/QT/CYP-perpetrator), checked for presence in
      DDInter, SNOMED substance, UNII, ATC. **Stop condition: < 70% DDInter
      coverage of the NTI/QT stratum means re-scope**, not push on. This is
      risk R1, the highest-scored risk in the plan.
- [ ] **Q3** — does DDInter 2.0 actually contain drug–food, drug–disease and
      therapeutic duplication content? I believe it is drug–drug only. Scope
      depends on the answer.
- [ ] **Q2 / Q22** — unpack a SNOMED release and check: does the India edition
      carry substance concepts of its own, and is there a **UNII map refset**?
      Both are `grep` questions. Q22 decides how load-bearing RxNorm is.
- [ ] **Q5 / Q6** — CredibleMeds and ONCHigh redistribution terms (legal).
- [ ] **Q24** — **ask to see the AIIMS drug master.** It's a spreadsheet, and
      looking at it collapses most of the remaining uncertainty in the plan:
      row count, coding level, FDC share.
- [x] **Affiliate Licence held, MLDS access in place, release files downloaded
      locally.** Phase 2 is unblocked. Remaining licensing item: confirm **CDCI**
      is available under the same registration — an International Edition
      download does not imply it.
- [ ] **Inspect the local RF2** — the recipe in
      [`snomed-releases/README.md`](snomed-releases/README.md) answers Q22
      (UNII map refset?), confirms the product/substance split, and sizes the
      `Is modification of` classification job. ~20 minutes.
- [ ] **Q27 — does CSNOServ's API support ECL?** Half a day against
      `nrces.in/bhts/api/v1/csnoserv/`. If yes, the Snowstorm standup (2.0
      eng-weeks) and the 16 GB build machine both disappear. Check the **API**;
      the browser at `/bhts/browser/` is CSNOFinder and works regardless.
- [ ] Licence memo signed by institutional legal.
- [ ] 16 GB build machine requested — **only if Q27 says Snowstorm is needed.**

## Conventions once code starts

| Area | Choice | Reference |
|---|---|---|
| Runtime service | Go, single binary, zero-allocation pair lookup over an mmap'd CSR index | `05` |
| Build ETL | Python, reproducible (`SOURCE_DATE_EPOCH`, sorted iteration, pinned digests) | `04` |
| Curation UI | Svelte | `01` Phase 2 |
| Curation store | PostgreSQL 16 | `02` |
| Auth | Keycloak OIDC, validated offline | `05 §8` |
| Terminology | Prefer **CSNOServ** (Apache-2.0 CSNOtk, Indian extensions pre-integrated); **BHTS** hosted for interactive curation; Snowstorm only as fallback. Build always consumes a checksummed ECL expansion cache from pinned RF2, never a live server | `12` |
| Artifact | Custom mmap'd flat file, Ed25519 detached signature | `04 §6` |

Effort baseline: **39–46.5 engineering person-weeks**, 7–9.3 clinical, 1.5–2.5
pharmacy, ~7 months calendar with two engineers (`07-effort.md`). The effort
table's subtotal is the sum of its rows — keep it that way when rows change.

## Style notes for this repo

- Plan documents prefer tables and schemas over prose, and state assumptions
  explicitly.
- Where a source fact is uncertain (DDInter's contents, CredibleMeds terms, the
  UNII refset), the documents say so and route it to a Phase 0 check rather than
  asserting it. Keep that habit — several of these are load-bearing.
- Cross-references between plan documents are relative Markdown links and are
  checked; if you move or rename a heading, fix the inbound links.
