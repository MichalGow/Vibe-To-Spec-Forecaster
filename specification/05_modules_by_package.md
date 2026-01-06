# 05 — Modules by Package (Responsibilities)

This section enumerates key modules and their responsibilities. Paths refer to `src/`.

## 1. `src/main.py`

- Defines CLI and dispatches to `_cmd_*` functions.
- Configures logging and `.env` loading.

## 2. `src/nations_seed/`

### `nations_seed/main.py`

Responsibilities:

- Scrape multiple Wikipedia-based nation lists:
  - largest by population
  - richest by GDP per capita (PPP)
  - most influential (power rankings)
- Normalize names using `pycountry`.
- Output a union list.

Outputs (legacy / timestamped):

- `data/<timestamp>_nations_seed/nations_seed.csv`
- `data/<timestamp>_nations_seed/nations_seed.json`
- `data/<timestamp>_nations_seed/README.md`

Note: the actively used seed for the rest of the pipeline is `data/nations_seed/seed.csv`.

## 3. `src/data_acquisition/`

### `data_acquisition/discovery.py` (`SourceDiscovery`)

Responsibilities:

- Discover sources for a nation and a time window.
- When Tavily is enabled:
  - runs multiple category-driven queries
  - filters out results whose dates appear to exceed `end_year_inclusive`
  - assigns `source_id` based on URL hash
- Always adds Wikipedia fallback pages per category.
- Adds native-language query expansions using `NATIVE_LANGUAGE_MAP`.

Category codes used by discovery:

- `A`: Global baseline datasets
- `B`: General overview
- `C`: Political/historical timeline
- `D`: Statistics/economy indicators
- `E`: Public mood/opinion
- `F`: Education/civic values
- `G`: Political speeches/manifestos
- `H`: Law/judiciary
- `I`: History/narrative
- `J`: Religion/moral framing
- `K`: Foreign relations / attitudes to others
- `L`: Militarism / violence norms
- `M`: Migration / demography

### `data_acquisition/downloader.py` (`DataDownloader`)

Responsibilities:

- Download a URL to disk.
- Infer extension from content type.
- Enforce minimum size per type.
- Write:
  - raw file named by `sha256(content)`
  - metadata sidecar `.json`
  - failure sidecar `FAILED_*.json` on errors or rejection.

### `data_acquisition/windowed_acquisition.py`

Responsibilities:

- Build windows using `src/utils/time_windows.build_five_year_windows`.
- For each window:
  - optionally acquire global sources into `data/raw_global_data/global/<window>`
  - acquire nation sources into `data/raw_nation_data/<ISO3>/<window>`
  - write manifests + collection report under `data/raw_nation_data/_meta/<window>/`.

### `data_acquisition/windowed_global_acquisition.py`

Responsibilities:

- Acquire global sources (Wikipedia window pages + global shock pages).
- Optionally acquire baseline datasets (`all_time`).

## 4. `src/utils/`

### `utils/text_extraction.py`

Responsibilities:

- Extract text from:
  - PDFs (first 20 pages) via `pypdf`
  - HTML via BeautifulSoup:
    - removes scripts/styles/nav/footer
    - tries to isolate main content (`#bodyContent`, `main`, `article`, etc.)

### `utils/time_windows.py`

- Defines `YearWindow` with `.label` `YYYY_YYYY`.
- Builds windows (end-exclusive).

### `utils/tavily_wrapper.py`

- `RateLimitedTavilyClient` with:
  - rate limiting (`RateLimiter`)
  - caching (`TTLCache`)
  - backoff + jitter

## 5. `src/processing/`

### `processing/llm_processor.py`

- Single OpenRouter client abstraction.

### `processing/windowed_mindset.py`

Responsibilities:

- For each `ISO3` and each window:
  - build evidence items from raw files (bounded excerpts)
  - split evidence into blocks by size
  - LLM summary per block
  - LLM synthesis into one state snapshot
  - validate citations, attempt auto-fix, write fixes json

Outputs are written under `data/processed_nation_data/<ISO3>/<window>/`.

### `processing/windowed_global_state.py`

Responsibilities:

- Same as windowed mindset, but for global evidence.

Outputs under `data/processed_global_data/global/<window>/`.

### `processing/process_mindset.py` (legacy, non-windowed)

- Aggregates processed source analyses into `data/processed/<ISO3>_mindset_analysis.md`.

### `processing/cross_nation_interactions.py`

- Loads `data/nations_seed/external_drivers.json`.
- Reads processed nation state snapshots.
- For each driver nation and window, asks LLM for bilateral interaction summary.
- Writes `interaction_context.json` + markdown summary.

## 6. `src/anonymise/`

### `anonymise/content_anonymiser.py`

- LLM-driven anonymisation rules:
  - replace country/city/party/leader/currency/etc with placeholders.

### `anonymise/facade_manager.py`

- Manages `data/facade/mapping.csv`.
- Creates new anon IDs like `Nation_A`, `Nation_B`, ...

### `anonymise/run_anonymisation.py`

- Reads `data/processed/<ISO3>_mindset_analysis.md`.
- Splits by `## SOURCE ID:` and anonymises each block.
- Writes `data/anonymised/<anon_id>_mindset.md`.

## 7. `src/forecast/`

### `forecast/forecaster.py`

- Generates JSON forecast from anonymised mindset doc.
- Writes `data/forecasts/<anon_id>_forecast.json`.

## 8. `src/forecasting/`

### `forecasting/windowed_forecast.py`

- Builds a forecast for a target window using:
  - historical nation state snapshots
  - historical global snapshots
  - optional interaction context
- Produces forecast markdown in a timestamped folder.
- Can produce an LLM-based evaluation markdown.

### `forecasting/scenario_forecaster.py`

- Scenario-based forecast as markdown (baseline/upside/downside) with probabilities.
- Writes to timestamped folder.

## 9. `src/evaluate/`

### `evaluate/evaluator.py`

- Numeric evaluation for non-windowed JSON forecasts.
- Scores numeric range predictions against ground-truth values.

### `evaluate/run_evaluation.py`

- Loads forecast and ground truth JSON.
- Writes `data/forecasts/<anon_id>_evaluation.json`.

## 10. `src/reporting/`

- PDF generator using ReportLab; see `08_reporting_pdf.md`.
