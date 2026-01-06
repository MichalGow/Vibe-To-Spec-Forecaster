# 08 — Reporting (PDF)

PDF reporting is implemented using ReportLab in `src/reporting/reportlab_report.py`.

## 1. Inputs

The PDF generator currently consumes:

- Forecast JSON:
  - `data/forecasts/<anon_id>_forecast.json`
- Nation raw acquisition artifacts (for source listing):
  - `data/raw_nation_data/<ISO3>/<window>/...` (sidecars are scanned to extract URLs)
- Facade mapping:
  - `data/facade/mappings.json` (ISO3 → anon_id)
- Seed file:
  - `data/nations_seed/seed.csv` (ISO3 → display name)

## 2. Output

The generator writes a timestamped output folder:

- `data/forecasts/<timestamp>_<anon_id>_<start>_<end>/forecast_report.pdf`

Where:

- `<start>_<end>` is derived from `forecast_year` and `step_years`.

## 3. CLI

Command:

```bash
./distiller.sh generate-pdf <ISO3> <forecast_year> --step-years 5 [--skip-url-validation]
```

Argument semantics:

- `forecast_year` is the end of the forecast window.
  - Example: `2030` with step 5 corresponds to window `2025_2030`.

## 4. PDF structure

Sections (current implementation):

- Cover page
  - report title
  - nation display name (see “Naming rules”)
  - report ID
  - generation date
  - historical data span label (currently `2000-<forecast_start>`)
  - sources cited count

- Executive Summary
  - uses `forecast_json.executive_summary` if present
  - prose may be LLM-polished

- Forecast Analysis
  - methodology paragraph
  - “Reasoning” section derived from the forecast JSON
  - scenario landscape
  - numeric forecasts table + rationales
  - stability & risks
  - black swan stress test

- Data Sources
  - sources grouped by category based on `source_id` category code

Conditional sections:

- Evaluation-related sections are **intended to be shown only if evaluation artifacts exist**.
  - In the current generator, evaluation is disabled by default for future forecasts.

## 5. Source extraction and URL validation

The report lists sources by scanning raw nation artifact sidecars:

- Traverse:
  - `data/raw_nation_data/<ISO3>/<window>/<source_id>/<date>/*.json`
- Read metadata fields:
  - `url`
  - `category`
  - `accessed_utc`

URL validation:

- If `--skip-url-validation` is NOT provided:
  - validate up to `max_url_checks` URLs (default 30).
  - validation uses HTTP `HEAD` then falls back to `GET`.

- If validation is skipped:
  - sources are marked unverified and the “Data Sources” text indicates that checks were skipped.

## 6. Parallel LLM polishing

To improve report quality, some sections are rewritten by the LLM.

- Implemented in `src/reporting/reportlab_llm.py`.
- Uses `ThreadPoolExecutor` (default max 4 workers).
- Each section is separately polished:
  - executive summary
  - reasoning
  - stability
  - black swan

## 7. Naming rules (dispute-safe display)

The PDF title uses `get_nation_name_from_seed(data_dir, iso3)`.

- Default: reads `data/nations_seed/seed.csv`.
- Overrides exist in code:
  - `TWN` is displayed as `Taiwan` regardless of seed wording.

This is implemented in `src/reporting/reportlab_data.py` (`DISPLAY_NAME_OVERRIDES`).
