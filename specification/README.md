# Knowledge Distiller (Blind Nation Forecasting) — Project Specification

This folder contains a **complete, code-derived specification** of the Knowledge Distiller project.

The goal of the system is to:

1. Acquire nation-level evidence from the public web.
2. Slice evidence into **strict historical time windows** (default 5-year windows, end-exclusive).
3. Convert raw evidence into **windowed “state snapshots”** using an LLM.
4. (Optionally) enrich forecasts with **cross-nation interaction context**.
5. Generate **forecasts for a future window** from historical windowed states.
6. (Optionally) evaluate forecasts against “actual” window state snapshots.
7. Produce a human-readable **PDF report** of a forecast.

The specification is intentionally **implementation-oriented**: file paths, CLI commands, artifact formats, and module responsibilities are described precisely so a developer can reproduce the system.

Important constraints and assumptions:

- The system is currently **file-artifact-based** (no DB layer in the current codebase).
- The system uses external providers:
  - **OpenRouter** for LLM calls
  - **Tavily** for web search (with Wikipedia-only fallback)
- This specification is derived from code + data artifacts.

## Contents

1. [`Entrypoints and CLI`](01_entrypoints_and_cli.md)
2. [`Architecture and Dataflow`](02_architecture_and_dataflow.md)
3. [`Data Contracts`](03_data_contracts.md)
4. [`External Services and Configuration`](04_external_services_and_configuration.md)
5. [`Modules by Package (Responsibilities)`](05_modules_by_package.md)
6. [`Windowed Pipeline Playbook (Recommended)`](06_windowed_pipeline_playbook.md)
7. [`Forecasting and Evaluation`](07_forecasting_and_evaluation.md)
8. [`Reporting (PDF)`](08_reporting_pdf.md)
9. [`Anonymisation and Blindness`](09_anonymisation_and_blindness.md)

## Glossary

- **ISO3**: A 3-letter country code (e.g., `DEU`). Used internally for acquisition and processing.
- **Anon ID**: A synthetic identifier like `Nation_A`. Used to hide identity from the forecaster.
- **Window label**: A time window string `YYYY_YYYY`, where the end year is **end-exclusive**.
  - Example: `2015_2020` means `[2015, 2020)`.
- **Source ID**: Stable identifier for a discovered source; used for citations and traceability.

## System invariants (must hold)

- **No leakage across time windows** in the windowed pipeline:
  - Acquisition and processing attempt to avoid evidence beyond the window end.
- **Every downloaded raw artifact has provenance**:
  - The raw file is stored and a `.json` sidecar stores URL, timestamps, checksum, etc.
- **LLM outputs are stored as files**:
  - State snapshots, forecasts, and evaluations are persisted as `.md` or `.json` artifacts.
