# Daily Stock Analysis API 配置与免费额度总结

---

# 8. 2026-08-07 当前 GitHub 配置核查与最终配置清单

## 8.1 核查结论

从 GitHub 截图可确认，目前已经建立 Gemini 主通道和 Groq 次通道，Groq 的协议、API Surface、Base URL 和模型 ID 均填写正确；当前仓库的 `00-daily-analysis.yml` 也已显式映射这些 `LLM_PRIMARY_*` 和 `LLM_SECONDARY_*` 变量，因此无须手动修改 workflow YAML。

但是当前设置存在一个需要修正的关键不一致：

```text
GEMINI_MODEL=gemini-3.5-flash
LLM_PRIMARY_MODELS=gemini-2.5-flash
```

启用以下设置后：

```text
LLM_CHANNELS=primary,secondary
```

项目会优先使用 Channels 配置。也就是说，真正的 Primary 模型由 `LLM_PRIMARY_MODELS` 决定，而不是 `GEMINI_MODEL`。按截图中的当前值，实际 Primary 仍会尝试 `gemini-2.5-flash`。

此前日志已显示 `gemini-2.5-flash` 曾返回 404，因此应把下面这个 Variable：

```text
LLM_PRIMARY_MODELS=gemini-2.5-flash
```

修改为：

```text
LLM_PRIMARY_MODELS=gemini-3.5-flash
```

修正后，实际路由才会变为：

```text
Gemini 3.5 Flash
    -> 失败时
Groq GPT-OSS 120B
```

Google 官方将 `gemini-3.5-flash` 标记为 Stable / GA，并支持 GenerateContent API；因此当前 `LLM_PRIMARY_API_SURFACE=` 可以保留。

## 8.2 当前 Variables 核查

### 可直接保留

```text
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-2.5-flash
GEMINI_REQUEST_DELAY=3.0

LLM_CHANNELS=primary,secondary

LLM_PRIMARY_PROTOCOL=gemini
LLM_PRIMARY_API_SURFACE=chat_completions
LLM_PRIMARY_ENABLED=true

LLM_SECONDARY_PROTOCOL=openai
LLM_SECONDARY_API_SURFACE=chat_completions
LLM_SECONDARY_BASE_URL=https://api.groq.com/openai/v1
LLM_SECONDARY_MODELS=openai/openai/gpt-oss-120b
LLM_SECONDARY_ENABLED=true

REPORT_TYPE=full
```

### 必须修改

```text
修改前：LLM_PRIMARY_MODELS=gemini-2.5-flash
修改后：LLM_PRIMARY_MODELS=gemini-3.5-flash
```

### 建议新增

```text
MAX_WORKERS=1
ANALYSIS_DELAY=10
SEARXNG_PUBLIC_INSTANCES_ENABLED=false
REPORT_LANGUAGE=zh-CN
REPORT_SHOW_LLM_MODEL=true
```

说明：

- `MAX_WORKERS=1`：避免 Gemini 和 Groq 免费层在短时间内被并发请求压满。
- `ANALYSIS_DELAY=10`：在连续分析股票之间增加间隔，降低 Groq TPM 和 Gemini 速率限制风险。
- `SEARXNG_PUBLIC_INSTANCES_ENABLED=false`：禁用此前频繁返回 403、418、429 和 500 的公共实例。
- `REPORT_LANGUAGE=zh-CN`：固定报告为简体中文。
- `REPORT_SHOW_LLM_MODEL=true`：在报告中显示实际使用的模型，便于确认是否触发 Groq fallback。

> 注意：当前上游 workflow 将 `GEMINI_REQUEST_DELAY` 固定写为 `3.0`，没有读取 GitHub Variable。因此截图中的 `GEMINI_REQUEST_DELAY=3.0` 与工作流默认值一致，但修改这个 Repository Variable 未必会改变运行值。若以后需要调整，需同步检查或修改 workflow 的环境变量映射。

## 8.3 当前 Secrets 核查

截图中已建立：

```text
ANSPIRE_API_KEYS
BOCHA_API_KEYS
BRAVE_API_KEYS
GEMINI_API_KEY
LLM_PRIMARY_API_KEY
LLM_SECONDARY_API_KEY
SERPAPI_API_KEYS
STOCK_LIST
TAVILY_API_KEYS
TICKFLOW_API_KEY
WECHAT_WEBHOOK_URL
```

用途与核查结果：

- `GEMINI_API_KEY`：保留，供 legacy Gemini 配置和兼容路径使用。
- `LLM_PRIMARY_API_KEY`：应填写与 `GEMINI_API_KEY` 相同的 Gemini Key，用于 Primary Channel。
- `LLM_SECONDARY_API_KEY`：应填写 GroqCloud 创建的 `gsk_...` Key，用于 Secondary Channel。
- `BRAVE_API_KEYS`：Brave 搜索主力之一。
- `TAVILY_API_KEYS`：AI 搜索主力之一。
- `SERPAPI_API_KEYS`：必须填写 SerpAPI Dashboard 的真实 Key，不能填写 endpoint URL。
- `BOCHA_API_KEYS`：中文新闻、港股公告和中文研报补强。
- `ANSPIRE_API_KEYS`：一次性免费点数用完前可保留，余额用完后可以删除。
- `TICKFLOW_API_KEY`：行情增强源，需在运行日志中进一步确认是否被实际调用。
- `STOCK_LIST`：自选股列表。
- `WECHAT_WEBHOOK_URL`：企业微信报告推送。

截图只能确认 Secret 名称已经创建，不能确认 Secret 内容是否正确。API 是否真正可用，仍需通过 GitHub Actions 日志和各服务 Dashboard 用量记录验证。

## 8.4 建议采用的最终 Variables

```text
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-2.5-flash
GEMINI_REQUEST_DELAY=3.0

LLM_CHANNELS=primary,secondary

LLM_PRIMARY_PROTOCOL=gemini
LLM_PRIMARY_API_SURFACE=chat_completions
LLM_PRIMARY_MODELS=gemini-3.5-flash
LLM_PRIMARY_ENABLED=true

LLM_SECONDARY_PROTOCOL=openai
LLM_SECONDARY_API_SURFACE=chat_completions
LLM_SECONDARY_BASE_URL=https://api.groq.com/openai/v1
LLM_SECONDARY_MODELS=openai/openai/gpt-oss-120b
LLM_SECONDARY_ENABLED=true

REPORT_TYPE=full
REPORT_LANGUAGE=zh-CN
REPORT_SHOW_LLM_MODEL=true
MAX_WORKERS=1
ANALYSIS_DELAY=10
SEARXNG_PUBLIC_INSTANCES_ENABLED=false
```

## 8.5 建议采用的最终 Secrets

```text
GEMINI_API_KEY=<Gemini API Key>
LLM_PRIMARY_API_KEY=<与 GEMINI_API_KEY 相同的 Key>
LLM_SECONDARY_API_KEY=<Groq gsk_... Key>

ANSPIRE_API_KEYS=<一次性额度用完前保留>
BOCHA_API_KEYS=<Bocha API Key>
BRAVE_API_KEYS=<Brave API Key>
SERPAPI_API_KEYS=<SerpAPI Dashboard 的真实 API Key>
TAVILY_API_KEYS=<Tavily API Key>
TICKFLOW_API_KEY=<TickFlow API Key>

STOCK_LIST=hk03416,hk00939,hk02638,KO,QYLD,JEPQ,SPCX,SMR,MU
WECHAT_WEBHOOK_URL=<企业微信机器人 Webhook>
```

## 8.6 当前 LLM 路由说明

Channels 模式开启后，建议把以下配置理解为两套独立通道：

### Primary Channel

```text
Protocol: gemini
API Surface: chat_completions
Model: gemini-3.5-flash
API Key: LLM_PRIMARY_API_KEY
```

### Secondary Channel

```text
Protocol: openai
API Surface: chat_completions
Base URL: https://api.groq.com/openai/v1
Model: openai/gpt-oss-120b
API Key: LLM_SECONDARY_API_KEY
```

Groq 官方将 `openai/gpt-oss-120b` 列为 Production Model。Free Plan 当前参考限额包括 30 RPM、1,000 RPD、8,000 TPM 和 200,000 TPD。限制按 Organization 统计，并以 GroqCloud 控制台的 Limits 页面为最终依据。

## 8.7 第一次验证步骤

1. 先把 `LLM_PRIMARY_MODELS` 修改为 `gemini-3.5-flash`。
2. 暂时将 `STOCK_LIST` 缩减为 `KO` 或 `hk00939,KO`。
3. 手动运行 GitHub Actions 的 full 或 stocks-only 模式。
4. 检查日志中的 Channels 配置是否显示 `primary,secondary`。
5. 检查 Analyzer 使用的主模型是否为 `gemini-3.5-flash`。
6. 若要单独验证 Groq，可临时设置 `LLM_PRIMARY_ENABLED=false`，运行一只股票。
7. 确认日志出现 `openai/gpt-oss-120b` 且报告成功生成。
8. 测试结束后恢复 `LLM_PRIMARY_ENABLED=true` 和完整 `STOCK_LIST`。
9. 在 GroqCloud Dashboard 检查 Requests、Input Tokens 和 Output Tokens 是否增加。
10. 在报告中确认实际模型名称，判断是否发生 fallback。

## 8.8 官方链接

- 项目仓库：https://github.com/ZhuLinsen/daily_stock_analysis
- 项目 Workflow：https://github.com/ZhuLinsen/daily_stock_analysis/blob/main/.github/workflows/00-daily-analysis.yml
- Gemini 3.5 Flash：https://ai.google.dev/gemini-api/docs/models/gemini-3.5-flash
- Gemini 模型目录：https://ai.google.dev/gemini-api/docs/models
- Google AI Studio：https://aistudio.google.com/
- Groq API Keys：https://console.groq.com/keys
- Groq 模型目录：https://console.groq.com/docs/models
- Groq OpenAI Compatibility：https://console.groq.com/docs/openai
- Groq Rate Limits：https://console.groq.com/docs/rate-limits
