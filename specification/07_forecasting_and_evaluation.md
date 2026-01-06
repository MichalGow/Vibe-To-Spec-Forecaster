# 07 — Forecasting and Evaluation

This project contains two forecasting/evaluation tracks:

1. **Non-windowed forecast (JSON)** from anonymised mindset.
2. **Windowed forecast (Markdown)** from historical state snapshots.

They are separate code paths.

## 1. Non-windowed forecast (JSON)

### Inputs

- `data/anonymised/<anon_id>_mindset.md`

Produced by `src/anonymise/run_anonymisation.py`.

### Generator

- `src/forecast/forecaster.py` (`Forecaster.generate_forecast(...)`)

### Output

- `data/forecasts/<anon_id>_forecast.json`

### Forecast JSON schema (as produced by the Forecaster prompt)

Required top-level keys:

- `nation_id` (string, anon ID)
- `forecast_date` (YYYY-MM-DD)
- `horizon_year` (int)
- `executive_summary` (string)
- `scenarios` (array)
- `numeric_forecasts` (array)
- `stability_forecast` (object)
- `black_swan` (object)

Scenario object:

- `name` (string)
- `probability` (float in [0,1])
- `description` (string)
- `key_drivers` (array of strings)

Numeric forecast object:

- `indicator` (string)
- `prediction_range` (string)
- `confidence` (`High`|`Medium`|`Low` as a string)
- `rationale` (string)

Stability forecast object:

- `trend` (string)
- `risk_factors` (array of strings)

Black swan object:

- `event` (string)
- `impact` (string)

## 2. Non-windowed evaluation (numeric, JSON)

### Inputs

- Forecast JSON:
  - `data/forecasts/<anon_id>_forecast.json`
- Ground truth JSON:
  - `data/ground_truth/<anon_id>_<horizon_year>.json`

### Evaluator

- `src/evaluate/evaluator.py` (`Evaluator.evaluate_forecast(...)`)

### Output

- `data/forecasts/<anon_id>_evaluation.json`

### Evaluation JSON schema

- `numeric_scores`: array of per-indicator items:
  - `indicator`
  - `predicted`
  - `actual`
  - `score`
- `overall_numeric_accuracy`: float
- `scenario_score`: optional object

Scoring rule for numeric ranges:

- Extract first two numbers from `prediction_range`.
- If `actual` is inside `[low, high]`: score `1.0`.
- Else: score `1.0 / (1.0 + distance_to_nearest_bound)`.

## 3. Windowed forecast (Markdown)

### Inputs

- Nation state snapshots:
  - `data/processed_nation_data/<ISO3>/<window>/<ISO3>_state.md`
- Global state snapshots:
  - `data/processed_global_data/global/<window>/global_state.md`
- Optional interaction context:
  - `data/processed_nation_data/<ISO3>/_interactions/interaction_context.json`

### Generator

- `src/forecasting/windowed_forecast.py`

Core behavior:

1. Determine anon ID via `data/facade/mappings.json`.
2. Load real `country_name` from `data/nations_seed/seed.csv`.
3. Load all nation windows ending before the forecast window start.
4. Load all global windows ending before the forecast window start.
5. Anonymize text:
   - replace ISO3 → anon_id
   - replace country name (case-insensitive) → `[Nation]`
6. Generate forecast markdown via LLM.
7. Persist forecast to timestamped folder.

### Output

- `data/forecasts/<timestamp>_<anon_id>_<start>_<end>/forecast.md`

## 4. Windowed forecast evaluation (LLM-based, Markdown)

### Inputs

- `forecast.md` in forecast folder
- Actual state for the forecast window:
  - `data/processed_nation_data/<ISO3>/<start>_<end>/<ISO3>_state.md`

### Generator

- `generate_forecast_evaluation(...)` in `src/forecasting/windowed_forecast.py`.

### Output

- `forecast-evaluation.md` in the same forecast folder.

The evaluation includes:

- overall accuracy score (0–100%)
- per-prediction scoring (Correct/Partial/Incorrect)
- category-level accuracy
- lessons learned

## 5. Scenario forecasting (Markdown)

### Generator

- `src/forecasting/scenario_forecaster.py`

Outputs:

- `scenario_forecast.md`
- optional `scenario_evaluation.md`

The scenario prompt requires probabilities sum to 100% and demands testable predictions per scenario.
