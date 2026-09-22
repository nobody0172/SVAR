# SVAR

## Install

Copy `SVAR.py` into your project. Python 3.8+, standard library only, no
dependencies.

```python
import SVAR
```

## Decision rule

The first condition that fires wins:

| # | Condition | Outcome |
|---|---|---|
| 1 | the proposal is already `UNKNOWN` | `UNKNOWN` |
| 2 | `V_specimen == 1` (explicit specimen clash) | `UNKNOWN` (veto) |
| 3 | structural score `< theta` | `UNKNOWN` |
| 4 | distributional evidence exists and similarity `< eta` | `UNKNOWN` |
| 5 | otherwise | retain the proposal |

Structural score is `1 - w * (V_unit + V_type + V_specimen)` when the proposed
concept has a source representative, and the flat prior `rho` when it does not.
Each verdict is `0` (compatible), `0.5` (undecidable) or `1` (explicit clash).
Only specimen carries a veto; unit and type feed the score.

`(w, rho, eta, theta)` are chosen once on a source-domain validation split and then
frozen — the same values apply to every target domain.

## Data shapes

Plain dicts. Only the keys below are read.

```python
field      = {"dataset": "A", "field_id": "f1", "field_label": "Serum Sodium",
              "source_table": "labs", "unit": "mEq/L", "data_type": "numeric",
              "specimen": "Blood", "n_rows": 120000}

label      = {"dataset": "A", "field_id": "f1", "gold_concept_id": "sodium"}   # or "UNKNOWN"

statistics = {"unit_group": "mEq/L",
              "quantiles": {"q05": 132.0, "q25": 136.0, "q50": 139.0,
                            "q75": 142.0, "q95": 147.0},
              "distribution_evidence_available": True}

profile    = {"concept_id": "sodium", "unit_group": "mEq/L",
              "quantiles": {...}}      # "scale" is derived if you omit it
```

`statistics` is optional per field. When it is absent, or marks itself unavailable,
or no profile matches its unit group, the distributional channel is simply skipped
for that field.

## Usage

### 1. Build source-only resources

```python
references = SVAR.build_references(train_fields, train_labels)
profiles   = SVAR.index_profiles(source_profiles)
units      = SVAR.UnitTable.load("unit_conversion.csv")   # optional
```

`build_references` picks one representative per concept: the labelled source field
with the most observations, then `(dataset, field_id)` for determinism. Only
source-*training* fields belong here.

### 2. Attach evidence to each frozen proposal

```python
evidence = [
    SVAR.verify(field, proposals[field["field_id"]],
                 references, profiles,
                 statistics.get(field["field_id"]), units)
    for field in fields
]
```

`verify` makes no network call and cannot change a proposal. Each returned row
carries `V_unit`, `V_type`, `V_specimen`, a printable `verification_rules` list, and
the distributional distance and score when they exist.

### 3. Select parameters on the source domain

```python
selection, trials = SVAR.calibrate(validation_evidence, validation_labels)
params = selection["parameters"]
```

`selection` also carries the counts and metrics the chosen point reached on the
validation split; `trials` holds every point that was searched.

Every declared `(w, rho, eta)` is crossed with every distinct structural score
observed on the validation split, plus accept-all and reject-all. Selection
maximises mapping-pair F1; ties break on higher `R`, lower `FAR_U`, lower `FAR_K`,
then declared order. Pass your own `grid` to override `SVAR.DEFAULT_GRID`.

`calibrate` requires a validation split containing both fields with a target and
fields confirmed to have none — an open-set threshold cannot be chosen otherwise.

### 4. Decide, then score

```python
rows   = SVAR.decide(evidence, params)
result = SVAR.evaluate(gold, rows)
```

Each row of `rows` carries `final_concept_id`, `decision` (`accept` / `abstain`) and
a `decision_reason` naming the rule that fired.

## Metrics

`evaluate` returns counts and four ratios in `[0, 1]`.

| | |
|---|---|
| `N_K` | fields with a target concept |
| `N_U` | fields confirmed to have no target |
| `C` | accepted and correct |
| `E_K` | accepted on a known field, wrong concept |
| `E_U` | accepted on a no-target field |
| `N_A` | `C + E_K + E_U` |

```
FAR_K = E_K / N_K        FAR_U = E_U / N_U
R     = C   / N_K        F1    = 2C / (N_K + N_A)
```

A wrong concept on a known field counts as both a false positive and a false
negative. An undefined ratio is returned as `None`, never as zero.

Because verification only retains or rejects, `R` is bounded above by the
proposal's own recall and cannot be raised by verification. Gains show up as a
lower `FAR_U` at a controlled cost in `R`.

## Unit table

`UnitTable` reads conversions from a CSV so every coefficient is auditable. No
factor is hard-coded in the module.

```csv
from_unit,to_unit,factor,offset,dimension,condition,source
degF,degC,0.5555555556,-32,temperature,,<where this factor comes from>
mmol/L,mEq/L,1,0,concentration,monovalent ions only,<where this factor comes from>
```

A row with a `condition` but no `factor` is reported as undecidable rather than
applied. The `mEq/L` ↔ `mmol/L` identity holds only for monovalent ions, so it is
applied only when the original labels on **both** sides name exactly one of sodium,
potassium, chloride or bicarbonate; otherwise the unit verdict falls back to `0.5`.

The table is optional. Without it, identical units are still compatible and units of
different known dimensions still clash, but convertible pairs become undecidable
instead of compatible.

## Bringing your own proposer

SVAR is agnostic to what produced the proposal — an LLM, a retriever, a lexical
matcher. It only needs a concept id or `UNKNOWN` per field.

For LLM proposers, `SVAR.PROPOSAL_SYSTEM_PROMPT`, `SVAR.proposal_schema(ids)` and
`SVAR.proposal_messages(field_view, catalog_view)` provide a ready prompt and a
strict JSON response schema. No request is issued; pass the messages to your own
client.

Pass `verify` the concept the proposer chose, unmodified. Freezing the proposal
before verification is what keeps the two stages separable and the verdicts
attributable.

## API

| | |
|---|---|
| `build_references(train_fields, train_labels)` | source structural representatives |
| `index_profiles(profiles)` | key profiles by `(concept_id, unit_group)` |
| `verify(field, concept, references, profiles, statistics=None, unit_table=None, types_are_declared=False)` | attach evidence |
| `structural_score(evidence, w, rho)` | structural score for one row |
| `decide(evidence_rows, parameters)` | retain or abstain |
| `calibrate(evidence_rows, validation_labels, grid=None)` | returns `(selection, trials)`; the chosen point is `selection["parameters"]` |
| `evaluate(gold, predictions)` | counts and the four metrics |
| `v_unit` / `v_type` / `v_specimen` | the individual verdicts |
| `UnitTable` / `normalize_unit` / `effective_unit` | unit handling |
| `specimen_class` | specimen normalisation |
| `distribution_evidence(field_quantiles, profile)` | `(distance, similarity)` |

Set `types_are_declared=True` only when both sides' types come from a schema
declaration. Types inferred from cast success rates misfire on numbers stored as
text, so an inferred clash stays undecidable.
