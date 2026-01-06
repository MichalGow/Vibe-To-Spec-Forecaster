# 01 — Entrypoints and CLI

This project is operated via a shell wrapper `distiller.sh` which invokes a Python CLI defined in `src/main.py`.

## 1. Shell entrypoint: `distiller.sh`

### Responsibilities

- Ensure `.venv` exists; if missing, create it (`python3 -m venv .venv`).
- Activate `.venv`.
- If `requirements.txt` exists, install dependencies (`pip install -r requirements.txt`).
- Set `PYTHONPATH` to include the repository root.
- Dispatch to `python3 -m src.main ...`.

### Supported wrapper commands

The wrapper supports aliases plus passthrough.

- `./distiller.sh`:
  - Runs `python3 -m src.main --help`.

- `./distiller.sh seed ...`:
  - Runs `python3 -m src.main nations-seed ...`

- `./distiller.sh acquire <ISO3> ...`:
  - Runs `python3 -m src.main data-acquisition --iso3 <ISO3> ...`

- `./distiller.sh acquire-all ...`:
  - Runs `python3 -m src.main data-acquisition ...`

- `./distiller.sh windowed-acquire ...`:
  - Runs `python3 -m src.main windowed-acquire ...`

- `./distiller.sh windowed-state ...`:
  - Runs `python3 -m src.main windowed-state ...`

- `./distiller.sh windowed-acquire-global ...`:
  - Runs `python3 -m src.main windowed-acquire-global ...`

- `./distiller.sh windowed-global-state ...`:
  - Runs `python3 -m src.main windowed-global-state ...`

- `./distiller.sh windowed-forecast ...`:
  - Runs `python3 -m src.main windowed-forecast ...`

- `./distiller.sh acquire-drivers ...`:
  - Runs `python3 -m src.main acquire-drivers ...`

- `./distiller.sh process-interactions ...`:
  - Runs `python3 -m src.main process-interactions ...`

- `./distiller.sh scenario-forecast ...`:
  - Runs `python3 -m src.main scenario-forecast ...`

- `./distiller.sh generate-pdf ...`:
  - Runs `python3 -m src.main generate-pdf ...`

- `./distiller.sh windowed <ISO3> [flags]`:
  - Convenience macro that runs:
    - `python3 -m src.main windowed-acquire --iso3 <ISO3> ...`
    - `python3 -m src.main windowed-state <ISO3> ...`
  - It splits flags into:
    - common: `--start-year`, `--end-year`, `--step-years`
    - acquisition-only: `--skip-global`, `--limit`
    - mindset-only: `--max-chars-per-file`, `--max-files-per-source`

- Default case:
  - Any other args are passed through to `python3 -m src.main ...`.

## 2. Python entrypoint: `src/main.py`

### Logging

`src/main.py` creates `logs/distiller_<timestamp>.log` and logs to both stdout and the file.

### CLI subcommands (authoritative)

The CLI is implemented with `argparse` and subcommands map 1:1 to `_cmd_*` functions.

- `nations-seed`
  - Calls `src.nations_seed.main.generate_nations_seed()`.

- `data-acquisition`
  - Args:
    - `--limit <int>`
    - `--iso3 <ISO3>`
    - `--targets <comma-separated ISO3 list>`
  - Calls `src.data_acquisition.main.run_data_acquisition(...)`.

- `process-mindset`
  - Args:
    - `<ISO3>`
  - Calls `src.processing.process_mindset.process_country_mindset(iso3)`.

- `anonymise`
  - Args:
    - `<ISO3>`
    - `<country_name>` (used only for anonymisation guidance)
    - `--capital <str>`
    - `--leaders <comma-separated str>`
  - Calls `src.anonymise.run_anonymisation.run_anonymisation(...)`.

- `forecast`
  - Args:
    - `<anon_id>`
    - `--horizon <int>` (default 2030)
  - Calls `src.forecast.run_forecast.run_forecast(anon_id, horizon)`.

- `evaluate`
  - Args:
    - `<anon_id>`
    - `--horizon <int>` (default 2030)
  - Calls `src.evaluate.run_evaluation.run_evaluation(anon_id, horizon)`.

- `windowed-acquire`
  - Args:
    - `--start-year <int>` (default 2000)
    - `--end-year <int>` (default 2025, end-exclusive)
    - `--step-years <int>` (default 5)
    - `--limit <int>`
    - `--iso3 <ISO3>`
    - `--skip-global`
    - `--wikipedia-only`
  - Calls `src.data_acquisition.windowed_acquisition.run_windowed_data_acquisition(...)`.

- `windowed-state`
  - Args:
    - `<ISO3>`
    - `--start-year`, `--end-year`, `--step-years`
    - `--max-chars-per-file` (default 30000)
    - `--max-files-per-source` (default 2)
  - Calls `src.processing.windowed_mindset.run_windowed_mindset(...)`.

- `windowed-acquire-global`
  - Args:
    - `--start-year`, `--end-year`, `--step-years`
    - `--no-baseline`
  - Calls `src.data_acquisition.windowed_global_acquisition.run_windowed_global_data_acquisition(...)`.

- `windowed-global-state`
  - Args:
    - `--start-year`, `--end-year`, `--step-years`
    - `--max-chars-per-file` (default 30000)
    - `--max-files-per-source` (default 2)
  - Calls `src.processing.windowed_global_state.run_windowed_global_state(...)`.

- `process-interactions`
  - Args:
    - `<ISO3>`
    - `--start-year` (default 2000)
    - `--end-year` (default 2020, end-exclusive)
    - `--step-years` (default 5)
  - Calls `src.processing.cross_nation_interactions.run_interaction_processing(...)`.

- `scenario-forecast`
  - Args:
    - `<ISO3>`
    - `<forecast_year>`
    - `--step-years` (default 5)
    - `--evaluate`
  - Calls `src.forecasting.scenario_forecaster.run_scenario_forecast(...)`.

- `acquire-drivers`
  - Args:
    - `<ISO3>`
    - `--start-year`, `--end-year`, `--step-years`
    - `--wikipedia-only`
  - Uses `get_primary_drivers(...)` then runs windowed acquisition for each driver.

- `generate-pdf`
  - Args:
    - `<ISO3>`
    - `<forecast_year>`
    - `--step-years` (default 5)
    - `--skip-url-validation`
  - Calls `src.reporting.pdf_report.run_pdf_generation(...)`.

- `pipeline`
  - Args:
    - `<ISO3>` `<country_name>`
    - `--capital`, `--leaders`
    - `--limit`
    - `--horizon` (default 2030)
    - `--force`
    - `--skip-acquisition`
    - `--skip-evaluation`
  - Orchestrates the non-windowed pipeline:
    1. `data-acquisition` (optional)
    2. `process-mindset`
    3. `anonymise`
    4. `forecast`
    5. `evaluate` (optional)
