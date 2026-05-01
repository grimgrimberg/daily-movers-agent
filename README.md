# Daily Movers Agent

[![CI](https://github.com/grimgrimberg/daily-movers-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/grimgrimberg/daily-movers-agent/actions/workflows/ci.yml)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

An agentic AI pipeline that fetches the day's market movers from Yahoo Finance, runs a 4-node LangGraph reasoning graph over each ticker, and produces explainable HTML / Excel / email digests. Cross-platform (macOS, Linux, Windows) and runs end-to-end with no API keys — LLM and SMTP are optional upgrades.

![Daily movers HTML digest — US most-active](digest-full-page.png)

---

## What it does

Each run is a five-stage pipeline:

1. **Ingest** — fetch a list of tickers (US "Most Active" screener, a regional universe, or a user watchlist).
2. **Enrich** — pull headlines (RSS), sector / industry, and a 15-day price series per ticker.
3. **Analyze** — LangGraph agent (Researcher → Analyst → Critic → Recommender), with raw OpenAI and deterministic heuristics as fallbacks.
4. **Review** — apply HITL rules (low confidence, big move, missing evidence) to flag rows for manual attention.
5. **Render** — write `digest.html`, `report.xlsx`, `digest.eml`, plus structured logs (`archive.jsonl`, `run.json`, `run.log`).

Per-ticker failures don't crash the run; every error is captured and the run reports `success` / `partial_success` / `failed`.

---

## Quickstart (2 minutes)

**macOS / Linux:**
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
python -m daily_movers run --mode movers --region us --top 5 --out runs/quick-test
```

**Windows (PowerShell):**
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt -r requirements-dev.txt
python -m daily_movers run --mode movers --region us --top 5 --out runs/quick-test
```

The HTML digest auto-opens in your browser. Run the test suite with:

```bash
python -m pytest -q
```

Helper scripts in `scripts/` cover bash quickstart, PowerShell tasks, and an Ethereal SMTP demo. See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for scheduling and email setup.

---

## Architecture

```
CLI / UiPath adapter
        │
        ▼
  Ingestion ──▶ Per-ticker (parallel) ──▶ Rendering
                ├─ Enrich
                ├─ Analyze (LangGraph → OpenAI → heuristics)
                └─ HITL review
```

The analysis core is a LangGraph `StateGraph` with four specialised nodes:

```
┌────────────┐   ┌──────────┐   ┌─────────┐   ┌─────────────┐
│ Researcher │──▶│ Analyst  │──▶│ Critic  │──▶│ Recommender │
└────────────┘   └──────────┘   └─────────┘   └─────────────┘
                      ▲              │
                      └─── retry ────┘   (low-confidence loop)
```

**Three-tier fallback** so the pipeline always produces output:

1. LangGraph agent (`langchain-openai`) — primary.
2. Raw OpenAI Responses API — secondary, used if LangGraph fails.
3. Deterministic heuristics — always available, no API key required.

Every ticker carries a `decision_trace` (evidence, numeric signals, rules triggered, summary) and provenance URLs, so analyst output is auditable rather than opaque.

Full architecture, ingestion details, schemas, and CLI reference live in [docs/TECHNICAL_OVERVIEW.md](docs/TECHNICAL_OVERVIEW.md).

---

## Multi-market watchlist

```bash
python -m daily_movers run --mode watchlist --watchlist watchlist.yaml --top 60 --out runs/watchlist
```

![Multi-market digest — watchlist across exchanges](multi-market-digest-full.png)

---

## Project layout

```
daily_movers/
  cli.py                   # argparse CLI
  config.py                # AppConfig + env loading
  models.py                # Pydantic models + HITL rules
  adapters/uipath.py       # UiPath function-call adapter
  email/                   # EML writer + optional SMTP backend
  pipeline/
    orchestrator.py        # main "do the whole run" function
    agent.py               # LangGraph 4-node StateGraph
    llm.py                 # raw OpenAI Responses API
    heuristics.py          # deterministic fallback
    critic.py              # guardrails (CoT removal, formatting)
  providers/               # Yahoo movers + per-ticker enrichment
  render/                  # HTML / Excel / EML renderers
  storage/                 # disk HTTP cache + structured logger

docs/                      # in-depth technical, deployment, security docs
specs/                     # design constitution, spec, plan, prompts
scripts/                   # quickstart, demo, Ethereal SMTP, UiPath smoke
tests/                     # unit + golden + adversarial tests
.github/workflows/ci.yml   # pytest on push / PR
```

---

## Documentation

- [docs/TECHNICAL_OVERVIEW.md](docs/TECHNICAL_OVERVIEW.md) — architecture, schemas, CLI reference, UiPath contract, testing
- [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) — environment variables, SMTP setup (Ethereal / Gmail / Mailtrap), scheduled runs
- [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) — debug recipes, VS Code launch configs, common issues
- [docs/OBSERVABILITY.md](docs/OBSERVABILITY.md) — structured logs, run metadata, suggested metrics
- [docs/BUSINESS_CONCEPT.md](docs/BUSINESS_CONCEPT.md) — problem framing and stakeholder view
- [docs/threat-model.md](docs/threat-model.md) — threat model and trust boundaries
- [docs/security-report.md](docs/security-report.md) — security posture summary
- [specs/](specs/) — design constitution, spec, execution plan, prompt templates

---

## License

[MIT](LICENSE)
