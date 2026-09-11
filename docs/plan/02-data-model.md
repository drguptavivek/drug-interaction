# 02 — Data Model

Two stores, deliberately different in kind:

| Store | Engine | Mutability | Purpose |
|---|---|---|---|
| **Curation store** | PostgreSQL 16 | Append-only log + rebuilt projections | Proposals, reviews, releases, provenance |
| **Runtime artifact** | Custom mmap'd flat file | Immutable, signed | Query-time lookup ([04](04-etl-pipeline.md#6-artifact-format)) |

Everything below is the curation store. Assumptions stated at the end.

## 1. Conventions

- `citext` for names, `text` for free text, `jsonb` for evidence blobs.
- Every table that records a human action carries `actor_sub` (Keycloak `sub`
  claim, immutable) *and* `actor_display` (name at the time), because Keycloak
  accounts get renamed and clinical audit needs the name as it was.
- All timestamps `timestamptz`, UTC.
- Surrogate keys are `bigint GENERATED ALWAYS AS IDENTITY` except the ingredient
  hub, which is discussed below.
- DDL below is ordered for **reading**, not for execution: several tables carry
  forward references (`proposal`, `licence`, `exception_code`). The migration
  orders them topologically. Nothing here depends on the exposition order.

## 2. Provenance

```sql
CREATE TYPE source_code AS ENUM (
  'DDINTER', 'SNOMED_INT', 'SNOMED_IN', 'CDC_INDIA', 'UNII',
  'ATC_WHO', 'OPENFDA', 'ONCHIGH', 'CREDIBLEMEDS', 'DRUGBANK_OPEN',
  'INSTITUTION'
);

CREATE TABLE source_release (
  id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  source         source_code NOT NULL,
  version        text        NOT NULL,   -- e.g. '2.0', '20250401'
  released_on    date,
  retrieved_at   timestamptz NOT NULL DEFAULT now(),
  retrieved_from text        NOT NULL,   -- URL or physical provenance
  sha256         char(64)    NOT NULL,
  licence_id     text        NOT NULL REFERENCES licence(id),
  redistributable boolean    NOT NULL,   -- decided in Phase 0, not guessed
  row_counts     jsonb       NOT NULL,   -- asserted at ingest, compared at rebuild
  UNIQUE (source, version)
);

CREATE TABLE licence (
  id           text PRIMARY KEY,          -- 'CC-BY-NC-SA-4.0', 'SNOMED-AFFILIATE', ...
  full_name    text NOT NULL,
  share_alike  boolean NOT NULL,
  non_commercial boolean NOT NULL,
  redistribution_allowed boolean NOT NULL,
  attribution_text text NOT NULL,         -- reproduced verbatim in artifact NOTICE
  retrieved_text text NOT NULL,           -- licence as retrieved, for the record
  retrieved_at timestamptz NOT NULL
);
```

`redistributable` is a hard gate: the artifact builder refuses to include any row
whose originating `source_release.redistributable` is false. That is how
[C2](11-challenges-to-the-brief.md#c2) (CredibleMeds) is enforced mechanically
rather than by memory.

## 3. The hub

```sql
CREATE TYPE ingredient_status AS ENUM ('draft','active','retired','merged');

CREATE TABLE ingredient (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  public_id     text NOT NULL UNIQUE,         -- 'ING-000417', minted by us, forever
  preferred_name citext NOT NULL,
  status        ingredient_status NOT NULL DEFAULT 'draft',
  merged_into   bigint REFERENCES ingredient(id),
  created_at    timestamptz NOT NULL DEFAULT now(),
  created_by    text NOT NULL,
  CHECK (status <> 'merged' OR merged_into IS NOT NULL),
  CHECK (merged_into IS DISTINCT FROM id)
);
CREATE UNIQUE INDEX ingredient_name_active
  ON ingredient (preferred_name) WHERE status = 'active';
```

`public_id` is the only identifier that appears in the runtime artifact's
provenance records and in release diffs. It must never be reused, and a merge
retires the loser rather than deleting it — a rule mapped away in release `r3`
must still be explicable in `r9`.

**Merges are the hard case.** Two ingredient rows minted separately and later
found to be the same moiety must merge without breaking prior releases. The
`merged_into` chain is resolved at artifact build time; the runtime artifact
contains only the survivor, but `ddictl diff` follows the chain so the diff
report says "ING-000417 merged into ING-000203" rather than "ING-000417 deleted".

## 4. Spokes

Each spoke is release-pinned and independently reviewable.

```sql
CREATE TYPE mapping_state AS ENUM ('proposed','approved','superseded','rejected');

CREATE TABLE spoke_snomed (
  id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  ingredient_id  bigint NOT NULL REFERENCES ingredient(id),
  sctid          bigint NOT NULL,
  module_id      bigint NOT NULL,          -- distinguishes Intl core vs India ext
  snomed_release_id bigint NOT NULL REFERENCES source_release(id),
  fsn_snapshot   text   NOT NULL,          -- FSN as at the pinned release
  semantic_tag   text   NOT NULL,
  is_moiety      boolean NOT NULL,         -- no outgoing 'Is modification of'
  state          mapping_state NOT NULL,
  proposal_id    bigint NOT NULL REFERENCES proposal(id),
  CHECK (semantic_tag = 'substance')       -- hard: never map to a product concept
);
CREATE UNIQUE INDEX spoke_snomed_one_active
  ON spoke_snomed (ingredient_id) WHERE state = 'approved';
CREATE UNIQUE INDEX spoke_snomed_sctid_unique
  ON spoke_snomed (sctid) WHERE state = 'approved';   -- no two moieties share an SCTID

CREATE TABLE spoke_ddinter (
  id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  ingredient_id  bigint NOT NULL REFERENCES ingredient(id),
  ddinter_id     text   NOT NULL,
  ddinter_name_snapshot text NOT NULL,
  ddinter_release_id bigint NOT NULL REFERENCES source_release(id),
  state          mapping_state NOT NULL,
  proposal_id    bigint NOT NULL REFERENCES proposal(id)
);
CREATE UNIQUE INDEX spoke_ddinter_one_active
  ON spoke_ddinter (ingredient_id) WHERE state = 'approved';
CREATE UNIQUE INDEX spoke_ddinter_id_unique
  ON spoke_ddinter (ddinter_id) WHERE state = 'approved';
```

The two partial unique indexes on `sctid` and `ddinter_id` are what make this a
hub-and-spoke and not a many-to-many mess. If a checker genuinely needs two
DDInter IDs on one moiety (DDInter listing a salt separately — one of the brief's
named exception classes), that goes in `spoke_ddinter_secondary`, which is
explicitly reviewed and explicitly flagged, never silently permitted:

```sql
CREATE TABLE spoke_ddinter_secondary (
  ingredient_id bigint NOT NULL REFERENCES ingredient(id),
  ddinter_id    text   NOT NULL,
  reason_code   text   NOT NULL REFERENCES exception_code(code),
  rationale     text   NOT NULL,
  proposal_id   bigint NOT NULL REFERENCES proposal(id),
  PRIMARY KEY (ingredient_id, ddinter_id)
);
```

## 5. Anchors

```sql
CREATE TYPE anchor_type AS ENUM ('UNII','ATC5','RXCUI');
CREATE TYPE spoke_side  AS ENUM ('snomed','ddinter');

-- Anchors are computed DOWN EACH SPOKE INDEPENDENTLY, then compared.
CREATE TABLE anchor_observation (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  ingredient_id bigint NOT NULL REFERENCES ingredient(id),
  side          spoke_side  NOT NULL,
  kind          anchor_type NOT NULL,
  value         text        NOT NULL,
  derivation    text        NOT NULL,   -- 'snomed refset 1119...', 'ddinter->drugbank->unii'
  source_release_id bigint NOT NULL REFERENCES source_release(id),
  UNIQUE (ingredient_id, side, kind, value)
);
```

Note the shape: `(ingredient, side, kind, value)` with no uniqueness on
`(ingredient, side, kind)`. That is deliberate — **ATC-5 is multi-valued**
([C10](11-challenges-to-the-brief.md#c10)). Agreement is then a computed view,
not a stored assertion:

```sql
CREATE VIEW anchor_agreement AS
WITH s AS (SELECT ingredient_id, kind, array_agg(value ORDER BY value) v
           FROM anchor_observation WHERE side='snomed'  GROUP BY 1,2),
     d AS (SELECT ingredient_id, kind, array_agg(value ORDER BY value) v
           FROM anchor_observation WHERE side='ddinter' GROUP BY 1,2)
SELECT COALESCE(s.ingredient_id, d.ingredient_id) AS ingredient_id,
       COALESCE(s.kind, d.kind)                   AS kind,
       s.v AS snomed_values, d.v AS ddinter_values,
       CASE
         WHEN s.v IS NULL OR d.v IS NULL           THEN 'absent'
         WHEN s.v && d.v                           THEN 'agree'      -- set overlap
         ELSE                                           'disagree'
       END AS verdict
FROM s FULL OUTER JOIN d USING (ingredient_id, kind);
```

`verdict = 'disagree'` on UNII is a **blocking** condition: the ranking tool will
not preselect a candidate, and the checker sees a red banner. `disagree` on ATC5
is advisory only, because ATC is assigned by indication and legitimately differs
between a substance-hierarchy view and a DDInter view.

## 6. Salt and product

```sql
CREATE TYPE collapse_decision AS ENUM (
  'collapse',        -- salt behaves as the moiety for interaction purposes
  'no_collapse',     -- prodrug/ester with a distinct profile; separate ingredient
  'collapse_flagged' -- collapses, but carries an exception predicate
);

CREATE TABLE salt (
  id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  sctid           bigint NOT NULL UNIQUE,   -- the salt substance concept
  fsn_snapshot    text   NOT NULL,
  moiety_ingredient_id bigint NOT NULL REFERENCES ingredient(id),
  -- provenance of the *proposal*, not the decision:
  derived_from_sctid bigint,                -- target of 'Is modification of'
  derivation_kind text,                     -- 'salt' | 'ester' | 'prodrug' | 'complex' | 'unknown'
  decision        collapse_decision NOT NULL,
  exception_code  text REFERENCES exception_code(code),
  rationale       text NOT NULL,
  proposal_id     bigint NOT NULL REFERENCES proposal(id),
  state           mapping_state NOT NULL
);
```

`derivation_kind` is the column that stops [C4](11-challenges-to-the-brief.md#c4)
from becoming a patient safety incident. `Is modification of` in SNOMED links
salts, esters, prodrugs and complexes alike. Collapsing all of them to the parent
moiety is wrong for the prodrug cases, where the interaction profile belongs to
the *conversion step*, not the parent. The ETL classifies `derivation_kind`
heuristically from FSN morphology and the ATC/UNII relationship, marks anything
it cannot classify as `unknown`, and **every row still goes through
maker-checker**. The brief's instruction to derive rather than hand-curate is
respected as far as it can safely go: the machine proposes 100% of rows, and a
human approves them.

```sql
CREATE TYPE route_class AS ENUM (
  'systemic_oral','systemic_parenteral','systemic_other',
  'topical','ophthalmic','otic','inhaled','intranasal',
  'rectal_local','vaginal_local','intrathecal','unknown'
);

CREATE TABLE product (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  source        source_code NOT NULL,        -- 'CDC_INDIA' or 'INSTITUTION'
  source_code_value text NOT NULL,
  sctid         bigint,                      -- if a SNOMED product concept exists
  name          text NOT NULL,
  dose_form_sctid bigint,
  route         route_class NOT NULL,
  is_fdc        boolean NOT NULL,
  status        mapping_state NOT NULL,
  proposal_id   bigint NOT NULL REFERENCES proposal(id),
  UNIQUE (source, source_code_value)
);

CREATE TYPE ingredient_relation AS ENUM (
  'has_active_ingredient',          -- 127489000  -> moiety level
  'has_precise_active_ingredient',  -- 762949000  -> salt level
  'has_basis_of_strength_substance' -- 732943007  -> what the strength refers to
);

CREATE TABLE product_ingredient (
  product_id    bigint NOT NULL REFERENCES product(id),
  ingredient_id bigint NOT NULL REFERENCES ingredient(id),
  salt_id       bigint REFERENCES salt(id),
  relation      ingredient_relation NOT NULL,
  strength_num  numeric,
  strength_unit text,
  basis_is_moiety boolean,   -- true = strength expressed as base, false = as salt
  PRIMARY KEY (product_id, ingredient_id, relation)
);
```

`basis_is_moiety` matters clinically and is routinely got wrong: 300 mg ferrous
sulphate is ~60 mg elemental iron. It is not used for pair lookup, but it is
displayed in finding text and it is needed the moment dose-aware rules are added.

## 7. Exceptions

```sql
CREATE TABLE exception_code (
  code        text PRIMARY KEY,     -- 'EXC_POLYVALENT_CATION', 'EXC_IRON_ROUTE', ...
  title       text NOT NULL,
  description text NOT NULL
);

CREATE TYPE exception_scope AS ENUM
  ('moiety_collapse','route_gate','salt_specific_rule','dose_form');

CREATE TABLE exception_rule (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  code        text NOT NULL REFERENCES exception_code(code),
  scope       exception_scope NOT NULL,
  predicate   jsonb NOT NULL,     -- see below
  effect      jsonb NOT NULL,     -- {"suppress":true} | {"severity":"major"} | {"require_salt_match":true}
  rationale   text  NOT NULL,
  reference   text,               -- citation
  proposal_id bigint NOT NULL REFERENCES proposal(id),
  state       mapping_state NOT NULL
);
```

Predicate grammar — deliberately tiny, total, and non-Turing-complete, so that it
can be evaluated in Go without a sandbox and reviewed by a clinician:

```jsonc
// Oral iron chelates levothyroxine; IV iron does not.
{
  "all": [
    {"ingredient": "ING-000417"},                     // iron
    {"route_in": ["systemic_parenteral"]},
    {"counterparty_ingredient": "ING-000088"}         // levothyroxine
  ]
}
// effect: {"suppress": true}
```

Supported node types: `all`, `any`, `not`, `ingredient`, `counterparty_ingredient`,
`route_in`, `counterparty_route_in`, `salt`, `counterparty_salt`, `dose_form_in`.
Nothing else. No arithmetic, no regex, no loops.

The brief's named exception classes map to seed rows:

| Exception | `code` | `scope` |
|---|---|---|
| Iron preparations | `EXC_IRON_ROUTE` | `route_gate` |
| Magnesium preparations | `EXC_MG_PREP` | `moiety_collapse` |
| Antacid polyvalent cations | `EXC_POLYVALENT_CATION` | `salt_specific_rule` |
| Oral vs parenteral iron | `EXC_IRON_ROUTE` | `route_gate` |
| Salt-specific DDInter entries | `EXC_SALT_SPECIFIC_DDI` | `salt_specific_rule` |
| *(added)* topical/ophthalmic/inhaled systemic exposure | `EXC_LOCAL_ROUTE` | `route_gate` |

## 8. Proposals and reviews — the append-only core

```sql
CREATE TYPE proposal_entity AS ENUM (
  'ingredient','spoke_snomed','spoke_ddinter','spoke_ddinter_secondary',
  'salt','product','product_ingredient','exception_rule',
  'alert_tier','suppression'
);

CREATE TABLE proposal (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity        proposal_entity NOT NULL,
  entity_key    text NOT NULL,          -- natural key, e.g. 'ING-000417/spoke_snomed'
  supersedes    bigint REFERENCES proposal(id),
  payload       jsonb NOT NULL,         -- the proposed row, validated against a JSON Schema
  evidence      jsonb NOT NULL,         -- ranked candidates + features actually shown
  tool_version  text NOT NULL,          -- ranking algorithm version, for reproducibility
  candidate_rank int,                   -- 1 = the tool's top pick; NULL = free-text entry
  candidate_score numeric,
  proposed_by   text NOT NULL,          -- Keycloak sub
  proposed_by_display text NOT NULL,
  proposed_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON proposal (entity, entity_key);

CREATE TYPE review_decision AS ENUM ('approve','reject','request_changes');

CREATE TABLE review (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  proposal_id   bigint NOT NULL REFERENCES proposal(id),
  decision      review_decision NOT NULL,
  comment       text,
  reviewed_by   text NOT NULL,
  reviewed_by_display text NOT NULL,
  reviewed_at   timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX review_one_terminal
  ON review (proposal_id) WHERE decision IN ('approve','reject');
```

### 8.1 Maker-checker, enforced in the database

```sql
CREATE OR REPLACE FUNCTION enforce_maker_checker() RETURNS trigger AS $$
DECLARE p_by text;
BEGIN
  SELECT proposed_by INTO p_by FROM proposal WHERE id = NEW.proposal_id;
  IF p_by IS NULL THEN
    RAISE EXCEPTION 'review references unknown proposal %', NEW.proposal_id;
  END IF;
  IF p_by = NEW.reviewed_by THEN
    RAISE EXCEPTION
      'maker-checker violation: % cannot review their own proposal %',
      NEW.reviewed_by, NEW.proposal_id
      USING ERRCODE = 'check_violation';
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER trg_maker_checker
  AFTER INSERT ON review
  DEFERRABLE INITIALLY IMMEDIATE
  FOR EACH ROW EXECUTE FUNCTION enforce_maker_checker();
```

This satisfies the brief's "server-side, not in the UI" requirement in the
strongest available form: it holds even against direct `psql` access, a buggy
service build, or a future second client. The Phase 2 exit criteria require a
test that calls the API as a self-approver and asserts a 409.

### 8.2 Append-only, enforced in the database

```sql
CREATE OR REPLACE FUNCTION deny_mutation() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION '% is append-only (attempted %)', TG_TABLE_NAME, TG_OP
    USING ERRCODE = 'insufficient_privilege';
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_proposal_immutable BEFORE UPDATE OR DELETE ON proposal
  FOR EACH ROW EXECUTE FUNCTION deny_mutation();
CREATE TRIGGER trg_review_immutable   BEFORE UPDATE OR DELETE ON review
  FOR EACH ROW EXECUTE FUNCTION deny_mutation();

REVOKE UPDATE, DELETE ON proposal, review FROM ddi_app;
```

Correcting a mistake means a new proposal with `supersedes` set. There is no
edit path. Retention: never purge; the tables are small (tens of thousands of
rows for phase 1).

## 9. Projections

`mapping_projection` is a real table, rebuilt by a deterministic function, never
written by hand. A materialized view would also work; a table is chosen so the
rebuild can be transactional, audited, and diffed against the previous content.

```sql
CREATE TABLE mapping_projection (
  ingredient_public_id text PRIMARY KEY,
  preferred_name  text   NOT NULL,
  sctid           bigint NOT NULL,
  sctid_fsn       text   NOT NULL,
  ddinter_id      text   NOT NULL,
  unii            text,
  atc5            text[],
  anchor_verdict  text   NOT NULL,
  salt_count      int    NOT NULL,
  approved_by     text   NOT NULL,
  approved_at     timestamptz NOT NULL,
  source_proposals bigint[] NOT NULL
);

CREATE OR REPLACE FUNCTION rebuild_projection() RETURNS void AS $$
BEGIN
  TRUNCATE mapping_projection;
  INSERT INTO mapping_projection
  SELECT ... FROM ingredient i
    JOIN spoke_snomed  ss ON ss.ingredient_id = i.id AND ss.state = 'approved'
    JOIN spoke_ddinter sd ON sd.ingredient_id = i.id AND sd.state = 'approved'
   WHERE i.status = 'active';
END $$ LANGUAGE plpgsql;
```

Determinism test (Phase 2 exit): run `rebuild_projection()` twice and on a
restored dump; assert identical content hashes.

## 10. Interaction rules and alert tiering

```sql
CREATE TYPE ddi_severity AS ENUM ('major','moderate','minor','unknown');
CREATE TYPE alert_tier   AS ENUM ('interruptive','passive','suppressed');

CREATE TABLE interaction_rule (
  id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  ingredient_lo  bigint NOT NULL REFERENCES ingredient(id),
  ingredient_hi  bigint NOT NULL REFERENCES ingredient(id),
  severity       ddi_severity NOT NULL,
  mechanism_id   bigint REFERENCES text_block(id),
  management_id  bigint REFERENCES text_block(id),
  source_release_id bigint NOT NULL REFERENCES source_release(id),
  source_ref     text,
  onchigh        boolean NOT NULL DEFAULT false,
  applicability  jsonb,          -- same predicate grammar as exception_rule
  CHECK (ingredient_lo < ingredient_hi),   -- canonical unordered pair
  UNIQUE (ingredient_lo, ingredient_hi, source_release_id)
);

-- Deduplicated interaction prose. DDInter repeats management text heavily;
-- interning it cuts the artifact string blob by roughly an order of magnitude.
CREATE TABLE text_block (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  sha256  char(64) NOT NULL UNIQUE,
  body    text NOT NULL,
  source_release_id bigint NOT NULL REFERENCES source_release(id)
);

-- Tier is an institutional clinical decision, versioned separately from the
-- knowledge. It goes through maker-checker like everything else.
CREATE TABLE alert_tier_decision (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  scope         text NOT NULL,      -- 'global' or an institution key
  ingredient_lo bigint REFERENCES ingredient(id),
  ingredient_hi bigint REFERENCES ingredient(id),
  severity_match ddi_severity,      -- rule-level fallback when the pair is NULL
  tier          alert_tier NOT NULL,
  rationale     text NOT NULL,
  proposal_id   bigint NOT NULL REFERENCES proposal(id),
  state         mapping_state NOT NULL,
  CHECK ((ingredient_lo IS NULL) = (ingredient_hi IS NULL)),
  CHECK (ingredient_lo IS NOT NULL OR severity_match IS NOT NULL)
);
```

**Coverage is explicit, not inferred.** This table is what makes
`no_interaction_data` answerable:

```sql
CREATE TABLE ingredient_coverage (
  ingredient_id bigint NOT NULL REFERENCES ingredient(id),
  source_release_id bigint NOT NULL REFERENCES source_release(id),
  is_covered    boolean NOT NULL,   -- present in the source's drug list at all
  PRIMARY KEY (ingredient_id, source_release_id)
);
```

A drug absent from DDInter's drug list is *not* a drug with zero interactions.
Without this table the service cannot tell the difference, and it will eventually
tell a clinician that a combination is clear when it has simply never heard of
one of the drugs.

## 11. Validation and suppression capture

```sql
CREATE TABLE department (
  id text PRIMARY KEY, name text NOT NULL, facility_id text NOT NULL
);

CREATE TABLE department_submission (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  department_id text NOT NULL REFERENCES department(id),
  raw_text      text NOT NULL,      -- exactly as submitted, never normalized in place
  rank_in_dept  int,
  submitted_by  text NOT NULL,
  submitted_at  timestamptz NOT NULL DEFAULT now(),
  resolved_product_id bigint REFERENCES product(id)
);

CREATE TYPE validation_expectation AS ENUM
  ('must_alert','must_not_alert','must_alert_interruptive');

CREATE TABLE validation_case (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  department_id text NOT NULL REFERENCES department(id),
  drug_a_text text NOT NULL,
  drug_b_text text NOT NULL,
  drug_a_ingredient_id bigint REFERENCES ingredient(id),
  drug_b_ingredient_id bigint REFERENCES ingredient(id),
  expectation validation_expectation NOT NULL,
  clinical_note text,
  submitted_by text NOT NULL
);
```

`raw_text` is retained verbatim. Departmental submissions arrive as
`"Tab. Amoxycillin 500 + Clav 125 BD"`; the normalization is lossy and must be
re-runnable against the original when the normalizer improves.

## 12. Releases

```sql
CREATE TYPE release_state AS ENUM ('assembling','built','signed','published','withdrawn');

CREATE TABLE kb_release (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  number      int NOT NULL UNIQUE,           -- r1, r2, ...
  version_string text NOT NULL UNIQUE,       -- 'ddinter2.0+snomedIN20250401+r7'
  state       release_state NOT NULL,
  built_at    timestamptz,
  built_by    text,
  artifact_sha256 char(64),
  signature   bytea,
  source_release_ids bigint[] NOT NULL,
  previous_release_id bigint REFERENCES kb_release(id),
  notes       text
);

CREATE TABLE release_member (
  release_id  bigint NOT NULL REFERENCES kb_release(id),
  proposal_id bigint NOT NULL REFERENCES proposal(id),
  PRIMARY KEY (release_id, proposal_id)
);

CREATE TYPE diff_change AS ENUM ('add','modify','retire','merge');

CREATE TABLE release_diff (
  release_id bigint NOT NULL REFERENCES kb_release(id),
  change     diff_change NOT NULL,
  entity     proposal_entity NOT NULL,
  entity_key text NOT NULL,
  before     jsonb,
  after      jsonb,
  clinical_significance text,   -- set by the release approver, not computed
  PRIMARY KEY (release_id, entity, entity_key)
);
```

A single approval mutates nothing outside `proposal`/`review`. `kb_release`
assembly is a separate, role-gated action (`ddi-release`), and the diff report is
generated before the release can move `built → signed`. `withdrawn` exists
because a released KB will eventually need to be recalled; sites must be able to
verify that the version they are running has not been withdrawn — see
[05](05-go-service.md#6-startup-validation) for how that is checked without
network access.

## 13. Formulary overlay (per institution, outside the curation store)

Overlays are authored per site and are *not* part of the signed KB. They are
plain YAML, checksummed, and echoed into the audit log at startup.

```yaml
overlay_version: "aiims-2026.1"
institution: "AIIMS New Delhi"
kb_compatibility: "ddinter2.0+snomedIN20250401+r7"

formulary:
  mode: restrict              # restrict | all
  include_products: [ "CDC-IN-10422", "CDC-IN-10511" ]

tiering:
  default_tier: passive
  interruptive:
    - { severity: major, onchigh: true }
    - { pair: ["ING-000088", "ING-000417"] }
  suppress:
    - { pair: ["ING-000201", "ING-000309"], rationale: "Cardiology, co-prescribed by protocol", approved_by: "..." }

unresolved_policy: surface    # surface | surface_and_warn  -- never 'drop'
```

`unresolved_policy` has no `drop` value. That is intentional and structural: the
brief requires unresolved drugs never be silently dropped, and the cheapest way
to guarantee that is to make the unsafe configuration unrepresentable.

## 14. Assumptions

1. PostgreSQL 16 is available institutionally. If not, the schema degrades to
   PostgreSQL 12 by replacing `GENERATED ALWAYS AS IDENTITY` and `citext` usage.
2. Keycloak issues a stable `sub` claim and the institution does not recycle
   accounts between people. If accounts *are* recycled, clinical audit is broken
   and that is an institutional finding, not a schema problem.
3. CDC-India product codes are obtainable in bulk. If they are only available via
   an online lookup, `product` is seeded from the institutional formulary master
   instead and CDC-India codes are attached opportunistically. **This is an open
   question — see [10](10-open-questions.md#q4).**
4. Phase 1 volumes: ~650 ingredients, ~2,000 salts, ~4,000 products,
   ~120,000 applicable interaction rules after restricting to mapped moieties.
   The schema is comfortable to roughly 100× these numbers.
