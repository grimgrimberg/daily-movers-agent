# Sample Run

A real, frozen run of the pipeline against the US most-active screener (top 5, no LLM key configured — pure deterministic / heuristic path). Captured 2026-05-01.

## Files

| File | What it is |
|------|------------|
| [`digest.html`](digest.html) | The single-file HTML digest with sortable table and inline styling |
| [`archive.jsonl`](archive.jsonl) | One JSON line per ticker — full structured payload |
| [`run.json`](run.json) | Run metadata: parameters, timings, status, summary |
| [`run.log`](run.log) | Structured JSONL logs (events, stages, latencies, retries) |

## Preview the HTML digest

GitHub doesn't render HTML files inline, but this repo publishes the digest via GitHub Pages:

> **<https://grimgrimberg.github.io/daily-movers-agent/examples/sample-run/digest.html>**

Or clone the repo and open `examples/sample-run/digest.html` locally.

## Reproduce

```bash
python -m daily_movers run --mode movers --region us --top 5 --out runs/my-run
```

Output ends up under `runs/my-run/` with the same six files (plus `report.xlsx` and `digest.eml`, which are git-ignored from this sample but are produced by every run).
