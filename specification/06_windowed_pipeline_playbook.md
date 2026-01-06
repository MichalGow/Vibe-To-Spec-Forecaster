# 06 — Windowed Pipeline Playbook (Recommended)

This is the recommended operational flow for generating forecasts without time leakage.

## 0. Prerequisites

- Python 3.10+
- `.env` with:
  - `OPENROUTER_API_KEY`
  - `OPENROUTER_API_MODEL`
  - `TAVILY_API_KEY` (unless using `--wikipedia-only`)

Run all commands via `./distiller.sh`.

## 1. Acquire global evidence (windowed)

Command:

```bash
./distiller.sh windowed-acquire-global --start-year 2000 --end-year 2025 --step-years 5
```

Behavior:

- For each window `YYYY_YYYY`:
  - write raw global artifacts under:
    - `data/raw_global_data/global/YYYY_YYYY/...`
  - write meta manifests under:
    - `data/raw_global_data/_meta/YYYY_YYYY/...`

## 2. Acquire nation evidence (windowed)

Command:

```bash
./distiller.sh windowed-acquire --iso3 DEU --start-year 2000 --end-year 2025 --step-years 5
```

Optional flags:

- `--skip-global` (do not acquire global sources during this step)
- `--wikipedia-only` (disable Tavily)

Behavior:

- For each window `YYYY_YYYY`:
  - discover sources using Tavily queries + wiki fallbacks.
  - download each source and write:
    - raw file
    - sidecar `.json`
    - failure sidecars where needed
  - write manifest + collection report to `data/raw_nation_data/_meta/<window>/`.

## 3. Generate nation window state snapshots

Command:

```bash
./distiller.sh windowed-state DEU --start-year 2000 --end-year 2025 --step-years 5
```

Key parameters:

- `--max-chars-per-file` (default 30000)
  - truncates each evidence excerpt.
- `--max-files-per-source` (default 2)
  - limits how many artifacts per source are included.

Outputs:

- `data/processed_nation_data/DEU/YYYY_YYYY/DEU_state.md`
- Plus evidence dumps and block summaries.

Hard constraints enforced by prompts:

- “Do not guess” if evidence is missing.
- “Only use information within the time window” (end-exclusive).
- “Citations must match the exact source ids.”

## 4. Generate global window state snapshots

Command:

```bash
./distiller.sh windowed-global-state --start-year 2000 --end-year 2025 --step-years 5
```

Outputs:

- `data/processed_global_data/global/YYYY_YYYY/global_state.md`

## 5. Optional: build interaction context

Command:

```bash
./distiller.sh process-interactions DEU --start-year 2000 --end-year 2020 --step-years 5
```

Outputs:

- `data/processed_nation_data/DEU/_interactions/interaction_context.json`

Notes:

- Interaction evidence is built from the already-generated `*_state.md` files.

## 6. Generate a forecast for the next window

Command:

```bash
./distiller.sh windowed-forecast DEU 2030 --step-years 5
```

Interpretation:

- Forecast window is `2025_2030`.
- Historical nation and global windows used are those ending **before 2025**.

Outputs:

- `data/forecasts/<timestamp>_<anon_id>_2025_2030/forecast.md`
- If `--evaluate` flag used:
  - `forecast-evaluation.md`

## 7. Generate a PDF report

Command:

```bash
./distiller.sh generate-pdf DEU 2030 --step-years 5 --skip-url-validation
```

Outputs:

- `data/forecasts/<timestamp>_<anon_id>_2025_2030/forecast_report.pdf`

Important: the PDF generator currently reads forecast JSON (`data/forecasts/<anon_id>_forecast.json`).
If your flow uses `windowed-forecast` (markdown), you need a compatible forecast JSON for PDF generation (see `08_reporting_pdf.md`).
