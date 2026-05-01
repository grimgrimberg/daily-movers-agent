# Deployment & Configuration

This guide covers running the pipeline outside of a quick local demo: scheduling, environment configuration, and SMTP setup for real email delivery.

---

## Local run

```bash
python -m daily_movers run --mode movers --region us --top 20 --out runs/<date>
```

Outputs live under `runs/<date-or-out>`.

## Scheduled run (Windows Task Scheduler)

1. Create a new task.
2. Action: start a program.
3. Program: `py`
4. Arguments: `-3 -m daily_movers run --mode movers --region us --top 20 --out runs/%DATE% --no-open`
5. Set environment variables in the task or via a `.env` file in the repo.

## Scheduled run (cron, macOS / Linux)

```cron
0 9 * * 1-5  cd /path/to/daily-movers-agent && /path/to/.venv/bin/python -m daily_movers run --mode movers --region us --top 20 --out runs/$(date +\%F) --no-open
```

Ensure the output directory is persisted if running in CI or scheduled jobs.

---

## Configuration

Configuration is a Pydantic model (`AppConfig`) in `daily_movers/config.py`. `load_config()` loads `.env` (with `override=True` to avoid stale Windows shell vars).

### Runtime tuning

| Variable | Default | Purpose |
|----------|---------|---------|
| `MAX_WORKERS` | 5 | Thread pool size for parallel ticker processing |
| `MAX_REQUESTS_PER_HOST` | 5 | Concurrent HTTP requests allowed per host |
| `REQUEST_TIMEOUT_SECONDS` | 20 | Yahoo HTTP timeout |
| `CACHE_DIR` | `.cache/http` | Disk cache location |
| `CACHE_TTL_SECONDS` | 1800 | Cache freshness window |
| `LOG_LEVEL` | INFO | Logging verbosity |

### OpenAI (optional)

| Variable | Default | Purpose |
|----------|---------|---------|
| `OPENAI_API_KEY` | — | Enables LLM analysis (pipeline works without it) |
| `ANALYSIS_MODEL` | `gpt-4o-mini` | OpenAI model name |
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | API base URL |
| `OPENAI_TIMEOUT_SECONDS` | 45 | OpenAI request timeout |

### SMTP (optional)

| Variable | Default | Purpose |
|----------|---------|---------|
| `SMTP_HOST` | `smtp.gmail.com` | SMTP server hostname |
| `SMTP_PORT` | 587 | STARTTLS port |
| `SMTP_SSL_PORT` | 465 | SSL fallback port |
| `SMTP_USERNAME` | — | SMTP login username |
| `SMTP_PASSWORD` | — | SMTP login password |
| `FROM_EMAIL` | — | Sender address |
| `SELF_EMAIL` | — | Recipient address |

Minimal `.env` example (LLM + email):

```env
OPENAI_API_KEY=...
SMTP_HOST=smtp.gmail.com
SMTP_USERNAME=...
SMTP_PASSWORD=...
FROM_EMAIL=me@example.com
SELF_EMAIL=me@example.com
```

---

## Email & SMTP setup

### How email works

- `digest.eml` is always written to the run folder (no configuration needed).
- SMTP sending only happens when you pass `--send-email` and SMTP credentials are configured.
- The SMTP backend tries STARTTLS first, then falls back to SSL.

### Option A: Ethereal Email (recommended for demos)

[Ethereal](https://ethereal.email/) is a free fake SMTP service. Emails are captured in a web inbox — nothing is delivered to real addresses.

**Step 1 — Create an Ethereal account**

1. Go to https://ethereal.email/create
2. Click **Create Ethereal Account**
3. Download the `credentials.csv` file (or copy the SMTP credentials shown on screen)

**Step 2 — Place `credentials.csv` in the project root**

The file Ethereal gives you looks like:

```csv
"Service","Name","Username","Password","Hostname","Port","Security"
"SMTP","Your Name","your.name@ethereal.email","yourpassword","smtp.ethereal.email",587,"STARTTLS"
"IMAP","Your Name","your.name@ethereal.email","yourpassword","imap.ethereal.email",993,"TLS"
"POP3","Your Name","your.name@ethereal.email","yourpassword","pop3.ethereal.email",995,"TLS"
```

> `credentials.csv` is git-ignored — it will **never** be committed.

**Step 3 — Run the Ethereal helper**

```bash
python scripts/ethereal_run.py
```

This will:
- Read SMTP credentials from `credentials.csv`
- Run a fresh movers pipeline (top 5, US most-active)
- Send the digest via Ethereal SMTP
- Open `digest.html` in your browser

**Step 4 — View the captured email**

1. Go to https://ethereal.email/login
2. Log in with your Ethereal username and password
3. Click **Messages** — your digest email will be there

The helper script also accepts `.env`-supplied SMTP values; if not configured there it falls back to `credentials.csv`.

### Option B: Gmail (App Password)

Set in `.env`:

```env
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SSL_PORT=465
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password
FROM_EMAIL=your_email@gmail.com
SELF_EMAIL=your_email@gmail.com
```

Then run with `--send-email`:

```bash
python -m daily_movers run --mode movers --region us --top 5 --send-email --out runs/email-test
```

### Option C: Mailtrap

```env
SMTP_HOST=sandbox.smtp.mailtrap.io
SMTP_PORT=587
SMTP_SSL_PORT=465
SMTP_USERNAME=<mailtrap_username>
SMTP_PASSWORD=<mailtrap_password>
FROM_EMAIL=alerts@example.test
SELF_EMAIL=alerts@example.test
```
