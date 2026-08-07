# Daily Stock Analysis API 配置与长期免费方案

> 核查日期：2026-08-07  
> 适用项目：[ZhuLinsen/daily_stock_analysis](https://github.com/ZhuLinsen/daily_stock_analysis)  
> 说明：免费额度、模型名称和计费规则可能调整，正式使用前应再次查看各服务控制台和官方定价页面。

## 1. Gemini 模型变量建议

### 1.1 图片中显示的信息

Google AI Studio 的用量页面显示曾调用以下两个模型：

- Gemini 2.5 Flash
- Gemini 3 Flash

错误统计同时出现：

- `404 NotFound`
- `503 ServiceUnavailable`

这与 GitHub Actions 日志一致：`gemini-3-flash-preview` 曾因高负载返回 503，而 `gemini-2.5-flash` 曾返回 404。

### 1.2 两个模型的准确 API 名称

- Gemini 2.5 Flash：`gemini-2.5-flash`
- Gemini 3 Flash：`gemini-3-flash-preview`

Gemini 2.5 Flash 是稳定模型；Gemini 3 Flash 目前仍属于 Preview 模型。Preview 模型通常可能具有更严格的速率限制，并可能较快被替换或弃用。

### 1.3 仅在这两个模型中选择时

建议将稳定模型设为主模型，将 Preview 模型作为备用：

```text
GEMINI_MODEL=gemini-2.5-flash
GEMINI_MODEL_FALLBACK=gemini-3-flash-preview
```

但是，之前的运行日志表明当前 API 项目调用 `gemini-2.5-flash` 时曾返回 404。因此修改后应先手动运行一次 GitHub Actions。若仍出现 404，这个组合不能作为可靠方案。

### 1.4 更推荐的长期配置

Google 当前模型目录已经提供更新的稳定 Flash 模型。若 Google AI Studio 的模型测试页能够成功调用 `gemini-3.5-flash`，建议使用：

```text
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-2.5-flash
```

如果 `gemini-2.5-flash` 仍持续返回 404，可改为另一个稳定且有免费层的轻量模型：

```text
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-3.5-flash-lite
```

建议避免继续让 `gemini-3-flash-preview` 作为唯一主模型。该模型可以暂时保留为第三候选，但当前项目的两个变量通常只能表达一个主模型和一个备用模型。

### 1.5 GitHub 中变量应放在哪里

`GEMINI_API_KEY` 是敏感信息，应放在：

```text
Settings -> Secrets and variables -> Actions -> Secrets
```

模型名称不是敏感信息，建议放在：

```text
Settings -> Secrets and variables -> Actions -> Variables
```

需要设置：

```text
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-3.5-flash-lite
GEMINI_REQUEST_DELAY=3.0
```

如果当前工作流只映射 Secrets 而没有映射 Repository variables，则应检查 `.github/workflows/00-daily-analysis.yml` 是否同时读取 `vars.GEMINI_MODEL` 和 `vars.GEMINI_MODEL_FALLBACK`。

### 1.6 官方链接

- 模型目录：https://ai.google.dev/gemini-api/docs/models
- 定价与免费层：https://ai.google.dev/gemini-api/docs/pricing
- Google AI Studio：https://aistudio.google.com/
- API Key 管理：https://aistudio.google.com/api-keys

---

## 2. API 总览

| 服务 | 分类 | 主要用途 | 免费额度概况 | 长期建议 |
|---|---|---|---|---|
| Gemini API | LLM | 生成个股分析、市场复盘和决策结论 | 部分模型免费输入和输出，但受项目级速率限制 | 保留，使用稳定模型并设置备用 |
| Brave Search API | 搜索 | 全球网页、新闻和美股资讯搜索 | 每月 5 美元信用额，Search 约可覆盖 1,000 次请求 | 长期主搜索源之一，注意超额计费 |
| Tavily API | 搜索 | 面向 AI Agent 的结构化网页和新闻搜索 | 每月 1,000 credits；Basic 1 credit/次，Advanced 2 credits/次 | 最优先新增 |
| SerpAPI | 搜索 | Google、Baidu 等搜索结果补充 | 免费计划每月 250 次搜索，每小时 50 次 | 保留作低频补充，确保填入真实 API Key |
| Anspire API | 搜索与 LLM | 联网搜索和模型聚合 | 新用户赠送一次性点数，不是固定月度额度 | 用完后停用，不作为长期核心 |
| Bocha API | 搜索 | 中文网页、新闻、公告和长摘要 | 免费政策以控制台为准；公开资料多为 1,000 次试用包 | 中文结果不足时再添加 |
| TickFlow API | 行情 | A股、美股、港股行情、日K和标的信息 | 历史日K免费服务无需 Key；注册 Free 套餐永久可用，具体实时权限以控制台为准 | 优先新增行情源 |
| Longbridge OpenAPI | 行情、基本面 | 港美股实时行情、历史行情、基本面和新闻 | 核心 API 免费，基础行情与账户权限相关 | 暂缓，能够开户后再添加 |
| SearXNG | 搜索 | 自建元搜索引擎和无配额兜底 | 开源免费；公共实例免费但不稳定 | 不自建时禁用公共实例 |

---

## 3. 各 API 详细说明

### 3.1 Gemini API

**用途**

- 生成股票综合分析报告
- 汇总技术面、新闻面和风险信息
- 生成市场复盘、评分、趋势和操作建议
- 在本项目中属于 LLM 层，不能被 Brave、Tavily 或 SerpAPI 替代

**免费额度**

Google 提供 Gemini Developer API 免费层，部分模型的输入和输出 token 免费，但具有模型级和项目级速率限制。免费层不是统一的“每月固定请求数”，具体限制应在 Google AI Studio 的 Rate limits 页面查看。同一 Google Cloud 或 AI Studio 项目创建多个 API Key，通常不会成倍增加项目额度。

**特点**

- 优点：免费层可持续、中文能力较好、项目原生支持
- 缺点：高峰期可能出现 503；Preview 模型稳定性较低；模型可用范围会变动
- 建议：主模型和备用模型均优先使用 Stable 版本

**项目变量**

```text
GEMINI_API_KEY=<真实密钥>
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-3.5-flash-lite
GEMINI_REQUEST_DELAY=3.0
```

**官网**

- https://ai.google.dev/gemini-api/
- https://ai.google.dev/gemini-api/docs/models
- https://ai.google.dev/gemini-api/docs/pricing
- https://aistudio.google.com/

### 3.2 Brave Search API

**用途**

- 全球网页和新闻搜索
- 美股、ETF、机构评级及英文资讯补充
- 在本项目中负责新闻、机构分析、风险排查等搜索任务

**免费额度**

官方 Search 价格为每 1,000 次请求 5 美元，每月自动赠送 5 美元信用额，因此理论上约覆盖 1,000 次 Search 请求。该模式属于“有免费月度信用额的计量计费”，不等于硬性免费套餐。账户可能需要支付方式，超额后可能产生费用。

**特点**

- 优点：独立搜索索引、英文及全球内容较好、速度快
- 缺点：9只股票每日多维搜索时，约1,000次/月可能接近上限
- 建议：保留为主搜索源，并在控制台设置用量提醒

**项目 Secret**

```text
BRAVE_API_KEYS=<真实密钥>
```

**官网**

- https://brave.com/search/api/
- https://api-dashboard.search.brave.com/documentation/pricing

### 3.3 Tavily API

**用途**

- 面向 LLM、RAG 和 AI Agent 的结构化搜索
- 提供实时网页、新闻、内容抽取和研究能力
- 适合美股、ETF、英文新闻和机构资讯

**免费额度**

Researcher 免费计划每月提供 1,000 API credits，无需信用卡。Basic Search 通常消耗 1 credit，Advanced Search 通常消耗 2 credits。Research、Crawl 和 Extract 可能消耗更多 credits。

**特点**

- 优点：每月免费额度明确；结果结构适合直接给 LLM；项目原生支持
- 缺点：如果项目使用 Advanced 或 Research 模式，实际请求数会低于1,000次
- 建议：当前最优先新增的搜索 API，与 Brave 分担搜索量

**项目 Secret**

```text
TAVILY_API_KEYS=<真实密钥>
```

**官网**

- https://www.tavily.com/
- https://www.tavily.com/pricing
- https://docs.tavily.com/documentation/api-credits

### 3.4 SerpAPI

**用途**

- 获取 Google、Baidu、Google News、Google Trends 等搜索引擎结果
- 适合高价值、低频的公告、处罚、诉讼和风险排查
- 作为 Brave 和 Tavily 的补充，而不是主要搜索源

**免费额度**

Free 计划为每月 250 次搜索，每小时吞吐上限 50 次。未使用额度通常按月重置。

**重要配置说明**

GitHub Secret 中必须填写 SerpAPI Dashboard 提供的真实 API Key，不是 endpoint。

正确：

```text
SERPAPI_API_KEYS=<SerpAPI Dashboard中的真实API Key>
```

错误：

```text
SERPAPI_API_KEYS=https://serpapi.com/search?engine=google
```

`https://serpapi.com/search?engine=google` 是接口地址的一部分，项目代码会自行拼接查询参数。

**特点**

- 优点：支持多种成熟搜索引擎；结果格式标准
- 缺点：免费额度较少；不适合承担全部股票的多维搜索
- 建议：修正 Key 后保留，控制为低频后备

**官网**

- https://serpapi.com/
- https://serpapi.com/pricing
- https://serpapi.com/search-api
- https://serpapi.com/baidu-search-api
- https://serpapi.com/manage-api-key

### 3.5 Anspire API

**用途**

- Anspire Search 联网搜索
- Anspire 模型服务和多模型聚合
- 本项目可同时将其用于搜索和 LLM 路由

**免费额度**

官网显示新用户赠送点数，但属于注册或活动赠送的一次性额度，不是明确的每月自动恢复免费额度。余额用尽后需要按量付费或充值。

**特点**

- 优点：中文接入方便；搜索和模型可以使用同一平台
- 缺点：免费额度不可持续
- 建议：在当前免费点数用完前使用，用完后从长期路线中移除

**项目 Secret**

```text
ANSPIRE_API_KEYS=<真实密钥>
```

**官网**

- https://open.anspire.cn/
- https://open.anspire.cn/document/docs/openPlatform/
- https://aisearch.anspire.cn/

### 3.6 Bocha API

**用途**

- 中文网页、新闻、公告和长摘要搜索
- 适合港股中文新闻、内地银行股、中文研报及公告补强
- 可提供适合 LLM 使用的结构化摘要

**免费额度**

官方开放平台提供免费接入或试用入口，但固定“每月自动重置”额度并不清晰。公开资料曾提到 1,000 次免费试用资源包，通常应视为一次性或限期资源包，而不是稳定的月度免费额度。最终以账户控制台显示为准。

**特点**

- 优点：中文内容和国内访问体验较好
- 缺点：长期免费政策不如 Tavily、SerpAPI 清晰
- 建议：只有在中文搜索明显不足时再添加

**项目 Secret**

```text
BOCHA_API_KEYS=<真实密钥>
```

**官网**

- https://open.bochaai.com/
- https://open.bochaai.com/pricing

### 3.7 TickFlow API

**用途**

- A股、美股和港股行情
- 历史日K、实时行情、分钟K线、标的信息和除权因子
- 改善 AkShare、Efinance 等免费爬虫接口的断连和限流问题

**免费额度**

TickFlow 提供两种免费路径：

1. 无需注册、无需 API Key 的免费服务，提供历史日K、周线、月线、季线、年线和标的信息，但不提供实时行情和分钟K线。
2. 注册后的 Free 套餐，官网称永久可用且无需信用卡，可体验实时行情和日K线；具体市场权限、频率和批量能力以账户控制台为准。

免费历史服务地址：

```text
https://free-api.tickflow.org
```

**特点**

- 优点：项目原生支持；针对行情数据；覆盖港股和美股；可减少免费网页接口波动
- 缺点：完整实时和批量能力可能受套餐权限限制
- 建议：当前最优先新增的行情 API

**项目 Secret**

```text
TICKFLOW_API_KEY=<真实密钥>
```

**官网**

- https://tickflow.org/
- https://docs.tickflow.org/zh-Hans/quickstart
- https://free-api.tickflow.org
- https://github.com/tickflow-org/tickflow

### 3.8 Longbridge OpenAPI

**用途**

- 港股、美股和A股基础行情
- 实时行情、历史K线、基本面、新闻和账户数据
- 解决港股名称和实时行情不完整问题

**免费额度**

官方说明交易、账户、基本面和新闻等核心 API 免费；基础 US LV1、HK LV1、CN LV1 行情可随符合条件的账户提供。高级港股 LV2、期权 OPRA 等行情可能收费。OpenAPI 行情权限与 App、PC 和网页端权限可能独立。

**特点**

- 优点：港美股针对性强；基础信息和实时行情较完整
- 缺点：需要账户、地区资格和 OpenAPI 授权
- 建议：当前无法开户则暂缓，以后具备条件后再添加

**项目配置示例**

```text
LONGBRIDGE_APP_KEY=<真实密钥>
LONGBRIDGE_APP_SECRET=<真实密钥>
LONGBRIDGE_ACCESS_TOKEN=<真实令牌>
```

也可按项目支持情况使用 OAuth 模式。

**官网**

- https://open.longbridge.com/
- https://open.longbridge.com/pricing
- https://open.longbridge.com/docs
- https://github.com/longbridge/openapi

### 3.9 SearXNG

**用途**

- 开源元搜索引擎
- 聚合多个搜索源
- 自建后可以作为无固定商业配额的搜索兜底

**免费额度**

软件本身开源免费，但自建需要服务器、网络和维护成本。公共实例通常免费，但可能出现 403、418、429、500、验证码或禁止 JSON 输出等问题。

**特点**

- 优点：自建时可控、隐私性好、无商业 API 次数套餐
- 缺点：公共实例极不稳定；自建需要运维
- 建议：当前不自建则禁用公共实例，避免拖慢分析

**项目变量**

```text
SEARXNG_PUBLIC_INSTANCES_ENABLED=false
```

如果以后自建：

```text
SEARXNG_BASE_URLS=https://你的SearXNG地址
```

**官网**

- https://docs.searxng.org/
- https://github.com/searxng/searxng

---

## 4. 推荐新增优先级

### 第 1 优先级：Tavily

原因：每月 1,000 credits、无需信用卡、项目原生支持，可在 Anspire 一次性额度用完后承担主要搜索任务。

```text
TAVILY_API_KEYS=<真实密钥>
```

### 第 2 优先级：TickFlow

原因：当前报告缺失不仅来自新闻，还来自港股名称、日K、实时价和行情源不稳定。TickFlow 比继续增加搜索 API 更能改善报告完整性。

```text
TICKFLOW_API_KEY=<真实密钥>
```

### 第 3 优先级：修正 SerpAPI

确认 GitHub Secret 填入真实 API Key，而不是 Google endpoint。

```text
SERPAPI_API_KEYS=<真实密钥>
```

### 第 4 优先级：修正 Gemini 模型

优先使用新版 Stable Flash 和 Stable Flash-Lite。先在 Google AI Studio 单独测试成功，再写入 GitHub Variables。

```text
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-3.5-flash-lite
```

### 第 5 优先级：Bocha

仅在港股中文资讯和中文公告不足时添加。免费资源包是否持续需以控制台为准。

```text
BOCHA_API_KEYS=<真实密钥>
```

### 以后再添加：Longbridge

目前无法开户则不配置。具备账户和 OpenAPI 权限后，可提升至行情源的高优先级。

---

## 5. 推荐的长期低成本组合

### LLM 层

```text
GEMINI_API_KEY=<真实密钥>
GEMINI_MODEL=gemini-3.5-flash
GEMINI_MODEL_FALLBACK=gemini-3.5-flash-lite
GEMINI_REQUEST_DELAY=3.0
```

### 搜索层

```text
BRAVE_API_KEYS=<真实密钥>
TAVILY_API_KEYS=<真实密钥>
SERPAPI_API_KEYS=<真实密钥>
SEARXNG_PUBLIC_INSTANCES_ENABLED=false
```

Anspire 余额未用完前可暂时保留：

```text
ANSPIRE_API_KEYS=<真实密钥>
```

余额用完后删除或停用。

### 行情层

```text
TICKFLOW_API_KEY=<真实密钥>
```

项目仍可保留 YFinance、AkShare 等内置免费源作为后备。Longbridge 暂不配置。

### 报告与运行参数

```text
REPORT_TYPE=full
REPORT_LANGUAGE=zh-CN
MAX_WORKERS=1
SEARXNG_PUBLIC_INSTANCES_ENABLED=false
```

---

## 6. 验证清单

下一次手动运行 GitHub Actions 后，建议在日志中确认：

- `Analyzer LLM` 显示新的 Gemini 主模型名称
- 不再出现 `gemini-2.5-flash is no longer available to new users`
- Tavily 显示已配置，并至少出现一次真实搜索结果
- TickFlow 显示已配置，而且实际日志出现 TickFlow 数据调用，不只是配置检查
- SerpAPI Dashboard 的 Searches Used 数值增加
- Brave Dashboard 的月度信用额消耗正常
- 港股 HK03416、HK00939、HK02638 的名称、日K或实时字段是否改善
- 最终成功分析数量是否从 5/9 提升

## 7. 风险提醒

- 搜索 API 不能替代 LLM API。即使 Brave、Tavily 和 SerpAPI 全部正常，Gemini 全部失败时仍可能无法生成最终决策报告。
- Brave 属于计量计费加月度信用额，应检查是否绑定自动扣费，并设置预算提醒。
- Gemini 免费层提交的内容可能用于改进 Google 产品，不应向模型发送账号密码、券商令牌或其他敏感信息。
- 免费额度、模型代码和权限会变化，建议每月或项目升级后复核一次官方定价和模型目录。
