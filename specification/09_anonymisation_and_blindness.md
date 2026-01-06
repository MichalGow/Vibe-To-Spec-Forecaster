# 09 — Anonymisation and Blindness

This project aims to test forecasting ability under partial blindness:

- The forecaster LLM should not know the nation identity.
- The system should allow evaluation against real outcomes while keeping forecast generation blind.

## 1. Two anonymisation tracks

### 1.1 Non-windowed anonymisation (LLM rewrite)

Implemented in `src/anonymise/`.

Inputs:

- `data/processed/<ISO3>_mindset_analysis.md`

Output:

- `data/anonymised/<anon_id>_mindset.md`

Mechanism:

- Split the input by `## SOURCE ID:` blocks.
- For each block:
  - LLM rewrites to remove:
    - country name
    - cities
    - parties
    - leaders
    - unique currency references
    - famous identifying events
  - Replaces with placeholders:
    - `[Nation]`, `[Capital]`, `[City]`, `[Leader]`, `[Party]`, `[Currency]`, `[Region]`, `[Neighbour]`

Retry behavior:

- If anonymised text is identical to original or still contains the country name:
  - retry up to 3 times.

## 2. Anon ID assignment

### 2.1 CSV mapping (managed by code)

`src/anonymise/facade_manager.py` manages:

- `data/facade/mapping.csv`

Properties:

- If missing, it is created.
- IDs are generated sequentially like:
  - `Nation_A`, `Nation_B`, ..., `Nation_Z`, `Nation_AA`, ...

Used by:

- `anonymise` CLI step
- `pipeline` command orchestration

### 2.2 JSON mapping (required by windowed forecasting/reporting)

Windowed forecasting and PDF reporting read:

- `data/facade/mappings.json`

This file is required by:

- `src/forecasting/windowed_forecast.py` (`load_facade_mapping` raises if missing)
- `src/reporting/reportlab_data.py` (returns ISO3 if missing)

Important: the current codebase does **not** write `mappings.json`.
To reproduce current functionality, a developer must implement one of:

- a generator that exports `mapping.csv` → `mappings.json`, or
- switch windowed forecasting/reporting to use `mapping.csv`, or
- maintain `mappings.json` manually.

## 3. Redaction rules in windowed forecasting

Windowed forecasting anonymizes by string replacement:

- ISO3 → anon_id
- country name (case-insensitive) → `[Nation]`

This occurs in:

- `src/forecasting/windowed_forecast.py` (`anonymize_text_for_forecasting`)

This is **not** as strong as LLM rewrite anonymisation but is designed for:

- hiding the most obvious identity-bearing tokens
- keeping state snapshots mostly intact

## 4. Evaluation and de-anonymization

Two evaluation modes exist:

- Numeric evaluation (non-windowed): compares forecast JSON to ground truth JSON.
- LLM evaluation (windowed): compares forecast markdown to actual state markdown.

In both cases, evaluation artifacts are intended to remain anonymized.

## 5. Operational safety guidance

To avoid identity leakage:

- Do not include real nation names in prompts passed to the forecaster.
- Ensure `data/processed_nation_data/<ISO3>/*_state.md` does not embed explicit nation name in plain text.
- Do not attach metadata sidecars directly into forecaster context.

## 6. Reporting name overrides

The PDF report uses a nation display name for readability.

- This is a presentation-layer concern.
- For disputed naming, apply overrides at rendering time (as implemented for `TWN`).
