# TechDistill 快速入门
TechDistill 通过聚合来自 GitHub、Hugging Face 和 Product Hunt 的趋势项目，并利用 AI 驱动的分析将其提炼为简洁的简报，助你将海量信息转化为核心洞察。

一份极简指南，助你在几分钟内启动 TechDistill 流水线。

---

## 前提条件

- 需要已经安装python，且python版本大于等于3.10.
- 需要配置以下网站提供的访问令牌：
   [Product Hunt](https://www.producthunt.com/v2/oauth/applications) API Token（`PH_API_TOKEN`）
   [GitHub](https://github.com/settings/tokens) 个人访问令牌（`GITHUB_TOKEN`）
   [Hugging Face](https://huggingface.co/settings/tokens) Token（`HF_TOKEN`）
- 配置ai提供商提供的api密钥：
   [OpenRouter](https://openrouter.ai/keys) API Key（`OPENROUTER_API_KEY`）—— 用于 AI 分析

- 可选（若在 GHA 运行则为必选）： 创建Telegram Bot 并获取 Telegram Bot Token 与 Chat ID —— 用于推送通知

以上令牌用于绕过平台速率限制（Rate Limits）并实现深层数据检索。AI 分析推荐使用支持 OpenAI 兼容格式的 Provider。
---

## 1. 安装依赖

```bash
# 创建虚拟环境（推荐）
python3.12 -m venv .venv
source .venv/bin/activate

# 安装依赖包
pip install -r requirements.txt
```

---

## 2. 配置环境

复制示例文件并填入你的密钥：

```bash
cp .env-example .env
```

`.env` 中最低要求的变量：

```env
PH_API_TOKEN=YOUR_PH_API_TOKEN_HERE
GITHUB_TOKEN=YOUR_GITHUB_TOKEN_HERE
HF_TOKEN=YOUR_HF_TOKEN_HERE

# AI 分析所需
OPENROUTER_API_KEY=YOUR_OPENROUTER_API_KEY_HERE
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
#选择低价付费模型
OPENROUTER_MODEL=deepseek/deepseek-v3.2

# 可选但推荐


# 可选：Telegram 推送，若托管到gha则为必选
TG_BOT_TOKEN=YOUR_Telegram_Bot_Token_HERE
TG_CHAT_ID=YOUR_Telegram_Chat_ID_HERE
```

> 完整的配置选项（概览设置、评论限制等）请参阅 `utils/config.py` 和 `.env-example`。


---

## 3. 运行流水线

```bash
# 默认运行：深度抓取 + AI 分析 + Telegram 监控
python main.py
```

### 常用 CLI 参数

| 参数 | 说明 |
|------|------|
| `--deep` / `--no-deep` | 启用或禁用深度抓取（默认：启用） |
| `--ai` / `--no-ai` | 启用或禁用 AI 分析（默认：启用） |
| `--watch` / `--no-watch` | 启用或禁用 Telegram 推送（默认：启用） |
| `--limit N` | 覆盖所有来源的抓取数量限制 |

### 示例

```bash
# 不带深度抓取和 AI 的快速运行
python main.py --no-deep --no-ai

# 限制每个来源抓取 3 条（快速测试）
python main.py --limit 3

# 生成报告但不推送 Telegram
python main.py --no-watch
```

---

## 4. 查看输出

每次运行都会在 `reports/` 下创建一个带时间戳的批次：

```text
reports/
└── TECH_PULSE_20260328_091045/
    ├── overview.md   # 每日 AI 生成概览
    ├── github.md     # GitHub Trending 报告
    ├── hf.md         # Hugging Face 报告
    └── ph.md         # Product Hunt 报告
```

打开任意 `.md` 文件即可阅读提炼后的简报。

---

## 5. 运行测试（可选）

```bash
python -m unittest discover -s test -p "test_*.py" -v
```

> 依赖网络的测试（例如 `test/test_first_token_latency.py`）需要手动运行。

---

TechDistill 采用针对 GitHub Actions 优化的无状态设计。通过利用 Serverless 编排和低成本的 LLM Token（如 DeepSeek-V3），我们将流水线的全年运行成本控制在 $3 以内。

## 6. 使用 GitHub Actions 自动化（可选）

本仓库包含 `.github/workflows/prism-pipeline.yml`。在仓库设置中配置所需的 secrets 后，流水线将按计划自动运行（默认：每天 UTC 06:00）或手动触发。

---

## 故障排除

| 问题 | 解决方案 |
|------|---------|
| `PH_API_TOKEN` 缺失错误 | Product Hunt 抓取需要此 Token。请从 Product Hunt 开发者后台获取。 |
| AI 分析被跳过 | 确保 `OPENROUTER_API_KEY` 已设置且有效。 |
| Telegram 未推送 | 检查 `TG_BOT_TOKEN` 和 `TG_CHAT_ID` 是否都已配置。 |
| 提供商错误 | 在选定模型的overview部分找到Providers部分，复制所需的Providers名称，在openrouter的setting部分找到privacy，并在privacy中的Providers选项添加所需的Providers。 |
| 隐私策略错误（此错误在使用免费模型时出现） | 在openrouter的setting部分找到privacy中的Data Policies部分，勾选含有Free endpoints的两个选项。|


| 问题 | 原因 |解决方法|
|------|---------|---------|
|LocalProtocolError错误|在 GitHub Actions（如 Azure 机房节点）并发调用 OpenRouter 免费 API 时，由于 API 速率限制（Rate Limiting）触发的网络协议异常。| 建议在生产环境或 CI/CD 流水线中切换至 低价付费模型 以获取更高的 QPS 限额。本地测试时可尝试使用免费模型，无需降低并发。 |


---

详细的架构说明、路线图和高级配置，请参阅 [README.md](README.md)。
