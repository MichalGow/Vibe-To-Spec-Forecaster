# 04 — External Services and Configuration

This system depends on external APIs and local configuration via environment variables.

## 1. Environment variables

The code loads environment variables from `.env` using `python-dotenv`.

### Required

- `OPENROUTER_API_KEY`
  - Used by `src/processing/llm_processor.LLMProcessor`.
  - If missing: raises `EnvironmentError`.

- `OPENROUTER_API_MODEL`
  - Used by `LLMProcessor`.
  - If missing: raises `EnvironmentError`.

- `TAVILY_API_KEY`
  - Used by `src/data_acquisition/discovery.SourceDiscovery` when not in Wikipedia-only mode.
  - If missing and not wikipedia-only: raises `EnvironmentError`.

### Optional

- `SITE_URL`
  - Used as OpenRouter `HTTP-Referer` header.
  - Default: `https://knowledge-distiller.local`.

- `APP_NAME`
  - Used as OpenRouter `X-Title` header.
  - Default: `Knowledge Distiller`.

### Present in repo but not used by current code

- `OLLAMA_EMBEDDING_MODEL`
  - Present in `.env` but there is **no embedding / Ollama integration** in `src/` at this time.

## 2. OpenRouter (LLM provider)

### Usage

All LLM calls flow through `src/processing/llm_processor.LLMProcessor`.

- Endpoint: `https://openrouter.ai/api/v1/chat/completions`
- Protocol: OpenAI-style chat completions.
- Headers:
  - `Authorization: Bearer <OPENROUTER_API_KEY>`
  - `HTTP-Referer: <SITE_URL>`
  - `X-Title: <APP_NAME>`

### `LLMProcessor.process_text(...)`

- Inputs:
  - `text` (string)
  - `prompt_template` (string) containing `{text}` placeholder
  - `system_message` (string)
  - `json_mode` (bool)

- Behavior:
  - Truncates `text` to 100k chars.
  - Escapes `{` and `}` in `text` before formatting into the template.
  - Sends a request with:
    - `temperature=0.1`
    - `max_tokens=4000`
  - If `json_mode=True`, requests `response_format={"type":"json_object"}`.
  - Retries up to 3 times with exponential backoff.

### LLM prompts

Prompts are embedded in:

- `src/processing/process_mindset.py` (raw → intelligence report)
- `src/processing/windowed_mindset.py` (evidence blocks → block summaries → state snapshot)
- `src/processing/windowed_global_state.py` (global evidence → global snapshot)
- `src/forecast/forecaster.py` (anon mindset → JSON forecast)
- `src/forecasting/windowed_forecast.py` (historical states → forecast markdown; evaluation markdown)
- `src/forecasting/scenario_forecaster.py` (scenario forecast markdown)
- `src/reporting/reportlab_llm.py` (polishing PDF prose)

## 3. Tavily (search provider)

### Wrapper

Tavily is used through `src/utils/tavily_wrapper.RateLimitedTavilyClient`.

Features:

- Process-wide leaky-bucket rate limiter (`src/utils/rate_limiter.py`).
- In-memory TTL cache (`src/utils/ttl_cache.py`).
- Retries (default max 5) on 429 and 5xx with exponential backoff + jitter.

### Wikipedia-only mode

`SourceDiscovery` supports `wikipedia_only=True`:

- No Tavily calls.
- Always uses deterministic Wikipedia fallback pages for key categories.

## 4. Network / I/O constraints

- All downloads are done via `requests.get(...)`.
- Downloaded HTML/PDF must exceed a minimum size threshold to be accepted:
  - PDF: 2000 bytes
  - HTML: 1000 bytes
  - JSON: 50 bytes

Failures are persisted as `FAILED_*.json` sidecars.

## 5. Security requirements

- `.env` must not be committed to version control (contains API keys).
- Forecasting must never pass real nation identity to the LLM.
- The “facade” mapping linking ISO3↔anon_id should be treated as sensitive.
