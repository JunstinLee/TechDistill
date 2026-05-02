# AGENTS.md — TechDistill

## Project identity
Python 3.10+ async pipeline that scrapes trending items from GitHub, Hugging Face, and Product Hunt, enriches them with detail, runs AI commentary via OpenRouter, and generates Markdown reports (optional Telegram push). Runs daily on GitHub Actions.

## Essential commands

```bash
# Install (virtual env recommended)
python3.12 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt

# Run pipeline
python main.py [--no-deep] [--no-ai] [--no-watch] [--limit N]

# Run tests (stdlib unittest—no pytest, no mypy, no formatter)
python -m unittest discover -s test -p "test_*.py" -v
```

Tests `test/test_first_token_latency.py` and `test/openrouter_raw_probe.py` need network + API keys; they are NOT matched by `test_*.py` — run them manually on demand.

## Architecture (what files do)

```
main.py                          # Async entrypoint, argparse, wires spiders → pipeline
core/pipeline.py                 # ScrapePipeline: orchestrates fetch → enrich → AI → report → Telegram
spiders/
  github_spider.py               # GitHub trending (HTML scrape + API enrich + README fetch)
  huggingface_spider.py          # HF API (models list + model card detail)
  producthunt_spider.py          # PH GraphQL API (posts list + detail)
utils/
  config.py                      # ALL config from env vars (dotenv). Single source of truth.
  ai_adapter.py                  # OpenRouterAdapter: streaming + non-streaming, diskcache, rate limiting
  reporter.py                    # MarkdownReporter: Jinja2 templates → reports/ dir
  overview_builder.py            # Builds highlight data + fallback text for overview
  overview_generator.py          # Calls AI (complete_async) to generate daily overview paragraph
  preprocessor.py                # TextCleaner, TokenManager: strip images/badges/TOC, truncate to 3k chars
  ai_output_sanitize.py          # strip <think> tags, collapse blank lines, length cap
  terminal_ui.py                 # Rich-based Live display for AI streaming progress
  telegram_notifier.py           # Sends Markdown reports as Telegram files (MD5 dedup)
  report_watcher.py              # watchdog observer; pushes only overview.md on file create/modify
  openrouter_rate_limiter.py     # Per-process semaphore + cooldown for OpenRouter API calls
  prompts/                       # AI system prompt templates (Markdown, {{variable}} placeholders)
  templates/                     # Jinja2 .j2 report templates
test/                            # unittest, naming: test_<module>.py, class <Module>Tests
```

### Spider contract
Every spider has:
- `source_name` class attr: `"github"`, `"hf"`, or `"ph"`
- `fetch_trending(limit)` → list of dicts with `name`, `desc`, `stats`, `url`, `raw_content`, plus an id key (`path`, `item_id`, or `slug`)
- `fetch_detail(id)` → dict containing `raw_content` (or `raw_readme`) that pipeline merges into list items

### Pipeline execution order
1. All spiders run sync (sequentially) — `run_task()` calls `fetch_trending` + optionally `fetch_detail`
2. AI analysis runs **async** — `asyncio.gather` with `Semaphore(3)` (max 3 concurrent calls)
3. Failed AI items (marked `❌`) are retried once after 15s delay
4. Overview built → overview AI generated → all Markdown reports written to `reports/TECH_PULSE_YYYYMMDD_HHMMSS/`
5. Telegram push (if `--watch`) sends each report file, skipping unchanged ones via MD5 state file

## Configuration (env vars)

All config lives in `utils/config.py` via `python-dotenv`. Copy `.env-example` to `.env`. Key vars:

| Var | Required? | Note |
|-----|-----------|------|
| `PH_API_TOKEN` | Yes for PH | Product Hunt GraphQL access |
| `GITHUB_TOKEN` | Recommended | Bypasses API rate limits |
| `HF_TOKEN` | Recommended | Bypasses HF rate limits |
| `OPENROUTER_API_KEY` | For AI | Missing key = AI analysis skipped |
| `OPENROUTER_MODEL` | For AI | Default `minimax/minimax-m2.5:free` |
| `OPENROUTER_BASE_URL` | For AI | Default `https://openrouter.ai/api/v1` |
| `TG_BOT_TOKEN` + `TG_CHAT_ID` | For Telegram | Both must be set for push |
| `OVERVIEW_MODEL` | For overview | Falls back to `OPENROUTER_MODEL` |
| `AI_COMMENT_MAX_CHARS` | Optional | Truncates AI comments (default 2000) |
| `AI_COMMENT_MAX_TOKENS` | Optional | Token cap per analysis call (default 768) |
| `OPENROUTER_STREAM_FALLBACK_TO_REASONING` | Optional | If stream content empty, use reasoning field (bool) |

## Git & CI

- **GitHub Actions**: `.github/workflows/prism-pipeline.yml` runs daily at 06:28 UTC (`cron: "28 6 * * *"`) + manual dispatch
- `.github/workflows/openrouter-first-token-latency.yml`: manual-only streaming benchmark
- CI requires GitHub Secrets: `PH_API_TOKEN`, `GH_TOKEN` (injected as `GITHUB_TOKEN`), `HF_TOKEN`, `OPENROUTER_API_KEY`, `TG_BOT_TOKEN`, `TG_CHAT_ID`, `OPENROUTER_CHAT_COMPLETIONS_EXTRA_JSON`
- **CRITICAL**: On CI, never use OpenRouter `:free` models — Azure runners get rate-limited to zero. Use a low-cost paid model.
- `.gitignore` covers `.env`, `reports/`, `__pycache__/`, `.venv/`, diskcache files

## Testing conventions

- Stdlib `unittest` only. No pytest fixtures. No coverage tooling configured.
- Every test file starts with `_ROOT = Path(__file__).resolve().parent.parent; sys.path.insert(0, str(_ROOT))`
- Tests use `unittest.mock.patch` extensively; no magic test framework
- `test/test_pipeline_pure.py` and `test/test_deep_mode.py` rely on `IsolatedAsyncioTestCase` for async pipeline tests
- Config-sensitive tests use `subprocess.run` in a child process to avoid polluting the parent process's already-loaded `Config`

## Gotchas & conventions

- **No type checker, no linter, no formatter configured** — Python is bare stdlib + pip deps only
- `asyncio.Semaphore(3)` governs max concurrent AI calls; there's an intentional `await asyncio.sleep(5)` between each call
- AI results are cached via `diskcache` (keyed by `source:name:content_hash:model`); `clear_cache()` flushes
- OpenRouter adapter uses `httpx.AsyncClient` with `http2=False` (HTTP/2 caused protocol errors)
- Rate limiter handles three constraints: per-minute window, min request interval (5s default), and 429 cooldown (30s default)
- Report watcher (`report_watcher.py`) is NOT used by `main.py` (which pushes directly after generation). The watcher is for standalone use only.
- Jinja2 templates intentionally skip AI comments that contain Chinese "信息不足" or "无法分析" or English "insufficient"/"not enough"
- When adding a new data source: create a spider with `source_name`, add it to the `tasks` list in `main.py`, create a prompt template in `utils/prompts/`, add mapping to `OpenRouterAdapter.TEMPLATE_MAP`, and update `reporter.py`'s `source_titles` and `source_key` loops
