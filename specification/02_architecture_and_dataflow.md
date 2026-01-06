# 02 — Architecture and Dataflow

## 1. High-level architecture

The system is a set of batch “pipeline blocks” implemented as Python modules under `src/`.

### Packages

- `src/nations_seed/`
  - Generates `data/nations_seed/seed.csv` (in repo) and legacy timestamped seeds.

- `src/data_acquisition/`
  - Discovers sources (Tavily + Wikipedia fallback).
  - Downloads raw artifacts and writes sidecar metadata.
  - Writes per-window manifests and collection reports.

- `src/processing/`
  - Converts raw artifacts into LLM-produced summaries and state snapshots.
  - The recommended mode is **windowed processing**.

- `src/anonymise/`
  - Produces anonymised “mindset” docs for the *non-windowed* pipeline.

- `src/forecast/`
  - Produces JSON forecasts from anonymised mindset docs (non-windowed pipeline).

- `src/forecasting/`
  - Windowed forecasting: predicts next window from historical window snapshots.
  - Scenario forecasting: baseline/upside/downside markdown outputs.

- `src/evaluate/`
  - Numeric evaluation (JSON) for non-windowed forecasts.

- `src/reporting/`
  - Generates PDF reports using ReportLab.

- `src/utils/`
  - Shared utilities: text extraction, time windows, Tavily wrapper, caching, rate limiting.

## 2. Dataflow overview (recommended windowed pipeline)

```
┌──────────────┐
│ seed.csv      │  data/nations_seed/seed.csv
└──────┬───────┘
       │
       v
┌───────────────────────────────────────────────┐
│ Windowed acquisition (raw_nation_data / raw_global_data)
│ - discover sources (Tavily + wiki fallbacks)
│ - download raw files + .json sidecars
│ - write per-window manifests
└──────┬────────────────────────────────────────┘
       │
       v
┌───────────────────────────────────────────────┐
│ Windowed processing (processed_nation_data / processed_global_data)
│ - build evidence excerpts
│ - split into blocks
│ - LLM block summaries
│ - LLM synthesis into {ISO3}_state.md / global_state.md
│ - citation validation/fix
└──────┬────────────────────────────────────────┘
       │
       v
┌───────────────────────────────────────────────┐
│ Optional: cross-nation interactions
│ - loads external_drivers.json
│ - summarizes bilateral relationships using processed states
│ - writes processed_nation_data/<ISO3>/_interactions/
└──────┬────────────────────────────────────────┘
       │
       v
┌───────────────────────────────────────────────┐
│ Windowed forecast
│ - loads historical nation states (pre-forecast window)
│ - loads historical global states (pre-forecast window)
│ - anonymizes (ISO3 + country name redaction)
│ - generates forecast.md in timestamped folder
│ - optional: forecast-evaluation.md
└──────┬────────────────────────────────────────┘
       │
       v
┌───────────────────────────────────────────────┐
│ PDF reporting
│ - loads forecast JSON for anon_id
│ - builds ReportLab PDF into timestamped folder
└───────────────────────────────────────────────┘
```

## 3. Non-windowed pipeline (legacy)

The `pipeline` command in `src/main.py` orchestrates a simpler set of artifacts:

1. `data/raw/` acquisition (not windowed)
2. `data/processed/<ISO3>_mindset_analysis.md`
3. `data/anonymised/<anon_id>_mindset.md`
4. `data/forecasts/<anon_id>_forecast.json`
5. `data/forecasts/<anon_id>_evaluation.json` (if ground truth exists)

This mode is still supported but the codebase’s newer functionality centers around **windowed acquisition/processing/forecasting**.

## 4. Persistence model (current state)

- There is **no database layer** in the current repository.
- All state is persisted as:
  - raw files and sidecar JSON
  - manifests and markdown reports
  - forecast JSON/markdown outputs
  - logs under `logs/`

## 5. Logging

- `src/main.py` configures a dual logger (stdout + `logs/distiller_<timestamp>.log`).
- Many modules also call `logging.basicConfig(...)` when executed standalone.

## 6. Key invariants

- **Window semantics**: end year is end-exclusive everywhere (`YYYY_YYYY`).
- **Provenance**: raw downloads produce metadata sidecars; failures also produce metadata.
- **Citations**: state snapshots contain citations like `[ISO3_G_<hash>]` and the pipeline attempts to validate/fix them.
