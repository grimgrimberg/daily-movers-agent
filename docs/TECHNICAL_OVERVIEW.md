# Technical Overview

The system is a Python CLI pipeline with an agentic analysis core (LangGraph) and deterministic fallbacks. It ingests market data, enriches each ticker, runs a multi-node reasoning graph, and renders multiple outputs. The design emphasizes explainability, consistent artifacts, and graceful degradation when external services fail.

---

## Pipeline overview

```
┌──────────────────────────────────────────────────────┐
│  CLI / UiPath Adapter                                │
│  ▼                                                   │
│  run_daily_movers(request, config)                   │
└──────────────────────────────────────────────────────┘
                         ▼
        ┌────────────────────────────────┐
        │  INGESTION                     │
        │  ├─ movers: Yahoo screener     │
        │  │   (JSON primary → HTML      │
        │  │    fallback)                │
        │  └─ watchlist: Chart API       │
        └────────────────────────────────┘
                         ▼
        ┌────────────────────────────────┐
        │  PER-TICKER (parallel)         │
        │  ├─ Enrich (headlines, sector, │
        │  │         price series)       │
        │  ├─ Analyze (LangGraph agent   │
        │  │   → OpenAI → heuristics)    │
        │  └─ HITL Review                │
        └────────────────────────────────┘
                         ▼
        ┌────────────────────────────────┐
        │  RENDERING & OUTPUT            │
        │  ├─ digest.html (auto-opens)   │
        │  ├─ report.xlsx                │
        │  ├─ digest.eml (always)        │
        │  ├─ SMTP send (if configured)  │
        │  ├─ archive.jsonl              │
        │  ├─ run.json                   │
        │  └─ run.log                    │
        └────────────────────────────────┘
```

Each run follows five stages:

1. **Ingest** a list of tickers (Yahoo "Most Active" screener, a curated regional universe, or a user watchlist).
2. **Enrich** each ticker with best-effort evidence (headlines, sector/industry, price series).
3. **Analyze** each ticker (LangGraph agent → raw OpenAI fallback → deterministic heuristics).
4. **Review** — apply HITL rules to flag items requiring manual attention.
5. **Render** — produce `digest.html`, `report.xlsx`, `digest.eml`, `archive.jsonl`, `run.json`, `run.log`.

Per-ticker failures don't crash the run, and every error is recorded.

---

## Modes: movers vs watchlist

### Movers mode

```bash
python -m daily_movers run --mode movers --region us --source most-active --top 20 --out runs/movers-demo
```

Answers: *"Given a market/region, what are the top N tickers worth investigating today?"*

- `region=us` → Yahoo "Most Active" screener (JSON primary, HTML fallback).
- `region=il|uk|eu|crypto` → curated universe ranking via chart data.

### Watchlist mode

```bash
python -m daily_movers run --mode watchlist --watchlist watchlist.yaml --top 60 --out runs/watchlist-demo
```

Answers: *"Analyze exactly these symbols."*

---

## Ingestion

Code: `daily_movers/providers/yahoo_movers.py`

### `--source` options (movers mode only)

| Source | Behavior |
|--------|----------|
| `auto` (default) | US → screener; non-US → universe ranking |
| `most-active` | Force Yahoo screener (only `region=us`) |
| `universe` | Force curated universe ranking (all regions) |

### Yahoo endpoints used

| Endpoint | Purpose | Fallback |
|----------|---------|----------|
| `v1/finance/screener/predefined/saved?scrIds=most_actives` | US most-active movers | → HTML scrape |
| `finance.yahoo.com/markets/stocks/most-active/` | HTML fallback for US movers | — |
| `v8/finance/chart/{symbol}` | Price history + non-US movers + watchlist | — |
| `feeds.finance.yahoo.com/rss/2.0/headline?s={symbol}` | Headlines | — |
| `finance.yahoo.com/quote/{symbol}` | Sector, industry, earnings | — |

### Watchlist file format (YAML/JSON)

```yaml
symbols:
  - AAPL
  - TEVA.TA
  - BP.L
  - ASML.AS
  - BTC-USD
```

- Symbols are uppercased, trimmed, deduplicated (order preserved).
- Empty/invalid items are silently ignored.
- If no valid symbols remain → `IngestionError`.

---

## Enrichment (best effort)

Code: `daily_movers/providers/yahoo_ticker.py`

Per ticker, enrichment attempts:
- **Headlines** (RSS feed — title + URL + published time)
- **Sector / Industry** (Yahoo quote page, regex extraction)
- **Earnings date** (best effort)
- **Price series** (last 15 closes from chart API, for sparkline)

Missing fields → `null`. Failures → recorded in `errors[]`. The run continues.

---

## Analysis architecture

### Agentic reasoning (LangGraph)

The core analysis uses a **LangGraph StateGraph** with four specialised nodes:

```
┌────────────┐     ┌──────────┐     ┌─────────┐     ┌─────────────┐
│ Researcher │────▶│ Analyst  │────▶│ Critic  │────▶│ Recommender │
└────────────┘     └──────────┘     └─────────┘     └─────────────┘
                        ▲               │
                        └── retry ◄─────┘  (conditional edge on low confidence)
```

| Node | Role |
|------|------|
| **Researcher** | Structures raw evidence from enrichment (headlines, numeric signals, sector) |
| **Analyst** | Produces sentiment, action (BUY/WATCH/SELL), confidence via `ChatOpenAI` or heuristic fallback |
| **Critic** | Guard-rails: CoT removal, confidence clipping, 2-sentence enforcement, provenance assembly |
| **Recommender** | Assigns portfolio tags: `top_pick_candidate`, `most_potential_candidate`, `contrarian_bounce`, `momentum_signal` |

Implemented in `daily_movers/pipeline/agent.py`.

### Three-tier fallback strategy

1. **LangGraph agent** (primary) — uses `langchain-openai` `ChatOpenAI`.
2. **Raw OpenAI** (secondary) — direct Responses API via `requests` (`daily_movers/pipeline/llm.py`).
3. **Deterministic heuristics** (always available) — rule-based, no API key needed (`daily_movers/pipeline/heuristics.py`).

With no `OPENAI_API_KEY`, the pipeline still runs and generates all artifacts.

### Required fields per ticker

| Field | Type | Constraint |
|-------|------|------------|
| `why_it_moved` | string | Exactly 2 sentences |
| `sentiment` | float | [-1, 1] |
| `action` | enum | BUY / WATCH / SELL |
| `confidence` | float | [0, 1] |
| `decision_trace` | object | evidence + numeric signals + rules + summary |
| `provenance_urls` | list | URLs backing the analysis |

Schema enforced by Pydantic models in `daily_movers/models.py`.

### Explainability

Each row includes a `decision_trace` with:
- evidence used (headlines)
- numeric signals
- rules triggered
- explainability summary

---

## HITL (human-in-the-loop) review rules

Implemented by `apply_hitl_rules()` in `daily_movers/models.py`.

Rows are flagged `needs_review=true` when any trigger fires:

- confidence < 0.75
- |% change| > 15
- missing headlines
- ingestion fallback used
- explicit errors exist

The row includes `needs_review_reason` listing all triggered reasons.

---

## Output artifacts (per run folder)

| File | Description |
|------|-------------|
| `digest.html` | Single-file HTML digest with inline styling and sortable table |
| `report.xlsx` | Spreadsheet report with highlights sheet |
| `digest.eml` | RFC822 email message (always generated, even without SMTP) |
| `archive.jsonl` | One JSON line per ticker — full structured data |
| `run.json` | Run metadata: parameters, timings, status, summary, email metadata |
| `run.log` | Structured JSONL logs (events, stages, errors, retries) |

The CLI prints `{status, summary, paths}` JSON to stdout after each run.

The HTML and Excel reports include open and close prices when available.

### Run status semantics

| Status | Meaning |
|--------|---------|
| `success` | All tickers processed without errors |
| `partial_success` | Artifacts generated, but ≥1 ticker had errors |
| `failed` | Run-level failure (rare) |

---

## CLI reference (every flag)

```bash
python -m daily_movers run [flags]
```

| Flag | Values | Default | Purpose |
|------|--------|---------|---------|
| `--date` | `YYYY-MM-DD` | today | Report metadata label (not historical) |
| `--mode` | `movers` / `watchlist` | `movers` | Ticker selection mode |
| `--region` | `us` / `il` / `uk` / `eu` / `crypto` | `us` | Region strategy (movers mode) |
| `--source` | `auto` / `most-active` / `universe` | `auto` | Movers ingestion source |
| `--top` | integer | 20 | Number of tickers to process |
| `--watchlist` | file path | — | Required for watchlist mode |
| `--out` | directory path | `runs/<date>` | Output directory |
| `--send-email` | flag | off | Attempt SMTP delivery |
| `--no-open` | flag | off | Don't auto-open digest.html |

`--date` sets report metadata only. It does not fetch historical data.

---

## UiPath integration (function-call adapter)

Module: `daily_movers/adapters/uipath.py`

```python
from daily_movers.adapters.uipath import run_daily_movers

result = run_daily_movers(
    out_dir="runs/uipath-demo",
    date="2026-02-09",
    mode="movers",
    region="us",
    source="most-active",
    top="20",
    send_email="false",
)
```

Contract:
- All inputs accept strings (UiPath-friendly coercion).
- Unknown arguments raise `TypeError`.
- `mode=watchlist` requires `watchlist` and the file must exist.
- Returns `dict` with keys: `status`, `summary`, `paths`.

---

## Testing

```bash
python -m pytest -q
```

### High-signal test subsets

| Command | What it covers |
|---------|---------------|
| `python -m pytest tests/test_models.py -q` | Pydantic schemas + HITL rules |
| `python -m pytest tests/test_yahoo_movers_source.py -q` | Ingestion routing + `--source` |
| `python -m pytest tests/test_uipath_adapter.py -q` | UiPath adapter contract |
| `python -m pytest tests/test_golden_run.py -q` | Artifact generation |
| `python -m pytest tests/test_email_backends.py -q` | Email backends + SMTP fallback |
| `python -m pytest tests/test_llm_normalization.py -q` | OpenAI output normalization |
| `python -m pytest tests/ralphing_harness.py -q` | Adversarial / failure paths |

`sitecustomize.py` sets `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` to prevent third-party pytest plugins from breaking test runs.
