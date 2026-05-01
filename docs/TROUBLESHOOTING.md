# Troubleshooting & Debugging

## VS Code debug configurations

The repo includes 4 debug configs in `.vscode/launch.json`:

- **Movers US (Most-Active Top 5 FAST)** — single-threaded, quickest for stepping through.
- **Movers US (Most-Active Top 20)** — full run.
- **Watchlist (All Exchanges 60)** — multi-market.
- **UiPath Adapter (Smoke)** — runs `scripts/uipath_smoke.py`.

## Practical debug recipe

```bash
# 1. Print resolved config
python -c "from daily_movers.config import load_config; print(load_config().model_dump())"

# 2. Run a small test
python -m daily_movers run --mode movers --region us --top 5 --out runs/debug-top5 --no-open

# 3. Inspect metadata and logs (Linux/macOS)
cat runs/debug-top5/run.json
tail -n 30 runs/debug-top5/run.log

# On Windows, use:
# Get-Content runs/debug-top5/run.json
# Get-Content runs/debug-top5/run.log | Select-Object -Last 30
```

## Where to look when something seems wrong

1. `run.json` — status, summary, email metadata.
2. `run.log` — structured JSONL with retries, latencies, errors.
3. `archive.jsonl` — full per-ticker payload including error details.

## Common issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Yahoo throttling | Too many parallel requests | Set `MAX_WORKERS=1` |
| Slow / timeout | Network latency | Raise `REQUEST_TIMEOUT_SECONDS` |
| No OpenAI output | Missing API key | Expected — heuristic fallback works |
| SMTP auth failure | Bad credentials | Non-fatal; `.eml` is still written |
| Run "looks empty" | Low `--top` or ingestion error | Check `run.json` + `archive.jsonl` |

## Cache behavior

- All HTTP calls go through a disk cache at `CACHE_DIR` (default `.cache/http`).
- TTL: `CACHE_TTL_SECONDS` (default 1800s).
- Force fresh data:
  ```bash
  rm -rf .cache/http              # macOS / Linux
  Remove-Item -Recurse -Force .\.cache\http   # Windows
  ```

## Reliability knobs for stable demos

```bash
# Linux / macOS
export MAX_WORKERS=1
export REQUEST_TIMEOUT_SECONDS=30
export OPENAI_TIMEOUT_SECONDS=60

# Windows (PowerShell)
$env:MAX_WORKERS=1
$env:REQUEST_TIMEOUT_SECONDS=30
$env:OPENAI_TIMEOUT_SECONDS=60
```
