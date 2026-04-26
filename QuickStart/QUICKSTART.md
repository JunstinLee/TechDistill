Here is the polished English version of your Quick Start guide, refined for clarity, professional tone, and technical idiomatic accuracy.

---

# TechDistill Quick Start Guide
TechDistill transforms the overwhelming flood of daily tech updates into actionable insights. By aggregating trending projects from **GitHub**, **Hugging Face**, and **Product Hunt**, and leveraging AI-driven analysis, it delivers concise, high-value briefings directly to you.

Follow this minimalist guide to get your TechDistill pipeline up and running in minutes.

---

## Prerequisites

- **Python**: Version 3.10 or higher.
- **Platform Access Tokens** (Required to bypass rate limits and enable deep data retrieval):
  - [Product Hunt](https://www.producthunt.com/v2/oauth/applications) API Token (`PH_API_TOKEN`)
  - [GitHub](https://github.com/settings/tokens) Personal Access Token (`GITHUB_TOKEN`)
  - [Hugging Face](https://huggingface.co/settings/tokens) Token (`HF_TOKEN`)
- **AI Provider Credentials**:
  - [OpenRouter](https://openrouter.ai/keys) API Key (`OPENROUTER_API_KEY`) — Powers the AI synthesis.
- **Optional (Required for GitHub Actions):**
  - A [Telegram Bot](https://core.telegram.org/bots#how-do-i-create-a-bot) Token and Chat ID for automated push notifications.

*Note: For AI analysis, we recommend using a provider that supports OpenAI-compatible formats.*

---

## 1. Installation

```bash
# Create a virtual environment (Recommended)
python3.12 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

---

## 2. Environment Configuration

Copy the example environment file and populate it with your keys:

```bash
cp .env-example .env
```

**Minimum required variables in `.env`:**

```env
PH_API_TOKEN=YOUR_PH_API_TOKEN_HERE
GITHUB_TOKEN=YOUR_GITHUB_TOKEN_HERE
HF_TOKEN=YOUR_HF_TOKEN_HERE

# AI Engine Configuration
OPENROUTER_API_KEY=YOUR_OPENROUTER_API_KEY_HERE
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
# Recommended: Use a cost-effective paid model for reliability
OPENROUTER_MODEL=deepseek/deepseek-v3.2

# Optional: Telegram Notifications (Required for GitHub Actions)
TG_BOT_TOKEN=YOUR_Telegram_Bot_Token_HERE
TG_CHAT_ID=YOUR_Telegram_Chat_ID_HERE
```

> **Pro Tip:** For advanced settings like summary length or comment limits, refer to `utils/config.py` and `.env-example`.

---

## 3. Running the Pipeline



Run the default pipeline (includes Deep Scraping, AI Analysis, and Telegram Notifications):

```bash
python main.py
```

### Common CLI Arguments

| Argument | Description |
|------|------|
| `--deep` / `--no-deep` | Enable/Disable deep scraping (Default: Enabled) |
| `--ai` / `--no-ai` | Enable/Disable AI analysis (Default: Enabled) |
| `--watch` / `--no-watch` | Enable/Disable Telegram push notifications (Default: Enabled) |
| `--limit N` | Override the maximum number of items retrieved per source |

### Examples

```bash
# Fast run without deep scraping or AI analysis
python main.py --no-deep --no-ai

# Quick test: Limit to 3 items per source
python main.py --limit 3

# Generate local reports without pushing to Telegram
python main.py --no-watch
```

---

## 4. Viewing Output

Every run generates a timestamped batch folder within the `reports/` directory:

```text
reports/
└── TECH_PULSE_20260328_091045/
    ├── overview.md   # Daily AI-generated executive summary
    ├── github.md     # GitHub Trending deep-dive
    ├── hf.md         # Hugging Face insights
    └── ph.md         # Product Hunt highlights
```

---

## 5. Testing (Optional)

```bash
python -m unittest discover -s test -p "test_*.py" -v
```

*Note: Network-dependent tests (e.g., `test/test_first_token_latency.py`) should be executed manually.*

---

## 6. Automation with GitHub Actions (Optional)

TechDistill is built with a **stateless architecture** optimized for GitHub Actions. By utilizing serverless orchestration and low-cost LLMs (like DeepSeek-V3), you can maintain this pipeline for **less than $3 USD per year**.

Locate `.github/workflows/prism-pipeline.yml` in the repository. Once you configure your secrets in GitHub Repository Settings, the pipeline will run automatically (Default: daily at 06:00 UTC) or can be triggered manually.

---

## Troubleshooting

| Issue | Resolution |
|------|---------|
| `PH_API_TOKEN` missing | Product Hunt scraping requires a token. Generate one in the Product Hunt Developer Dashboard. |
| AI analysis skipped | Verify `OPENROUTER_API_KEY` is correctly set and has remaining credit. |
| Telegram notification failed | Ensure both `TG_BOT_TOKEN` and `TG_CHAT_ID` are valid. |
| Provider Error | In the OpenRouter model overview, check the "Providers" section. Add the specific provider to your "Allowed Providers" list in OpenRouter's privacy settings. |
| Privacy Policy Error (Free Models) | In OpenRouter Settings > Privacy > Data Policies, ensure the two checkboxes for "Free endpoints" are enabled. |

### Technical Edge Cases

| Error | Cause | Resolution |
|------|---------|---------|
| `LocalProtocolError` | Triggered by Rate Limiting when calling free OpenRouter APIs concurrently (common on GHA nodes like Azure). | **Recommended:** Switch to a low-cost paid model for higher QPS. For local testing, you may continue using free models as concurrency is usually lower. |

---

For detailed architecture diagrams, development roadmaps, and advanced configurations, please refer to [README.md](README.md) and [DOCS/PROJECT.md](DOCS/PROJECT.md).