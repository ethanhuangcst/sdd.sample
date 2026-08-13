## 国家 / 地区限制

可用性可能变化；不同国家和地区对于技术栈的限制

### 大陆地区+香港

| 不可用 / 不稳定 | 替代方案 |
| --- | --- |
| **OpenAI（ChatGPT API）**（`api.openai.com`） | **DashScope（通义千问）** — OpenAI 兼容 + 原生图像 API |
| **npmjs.org** 注册源（慢或被拦） | **npmmirror** — `https://registry.npmmirror.com` |

### 仅中国大陆地区

| 不可用 / 不稳定 | 替代方案 |
| --- | --- |
| **Google Maps Platform**（客户端/服务端直连） | **高德地图（AMap）** · 或仍需 Google 数据时用 **Google Maps MCP for China**（`GMAPS_MCP_URL` / `GMAPS_MCP_BEARER`） |

说明：

- 香港一般可直连 Google Maps；高德 / Maps MCP 主要用于中国大陆。
- 面向中国大陆或香港的项目，默认 LLM 优先使用 Qwen DashScope。


## 技术栈与依赖(大陆与香港通用)

跨项目共用。密钥清单亦见 [`keys-cn.md`](./keys-cn.md)；运行时密钥仅写入 `.env.local`——切勿将真实密钥或 `.env.local` 提交到公共仓库。产品行为写在各项目的需求文档中，不在本文。

面向**中国大陆**与**香港**；优先选用在这些地区可用的服务。

### 技术栈

| 类别 | 技术 | 版本 |
| --- | --- | --- |
| 框架 | Next.js（App Router） | 16.3.0 |
| UI / 语言 | React · TypeScript | 19.2.8 · 7.0.2 |
| 样式 | Tailwind CSS | 4.3.1 |
| 状态 / 表单 | React Query · Zustand · RHF + Zod + `@hookform/resolvers` | 5.101.4 · 5.0.15 · 7.85.0 + 4.4.3 + 5.7.1 |
| 存储 | 本地 JSON（`data/`）— 无数据库 | — |
| AI | `openai` SDK → 通义千问 DashScope compatible-mode | 6.39.0 |
| 测试 | Vitest · RTL · Playwright | 4.1.9 · 16.3.2 · 1.61.1 |

规则：尽量少依赖；优先平台与标准库；**禁止数据库**（除非项目规格另有说明，否则仅使用 `data/` 下本地 JSON）。只接入产品实际需要的第三方能力——不要默认把下文所有服务都接上。

### 最小依赖策略

- 尽量少用依赖——优先平台能力、标准库与现有技术栈，再考虑新增包。
- 保持架构干净：不为「以防万一」引入可选库。

### npm 安装

在**中国大陆**与**香港**，请通过阿里云 npmmirror 安装包：

```bash
npm config set registry https://registry.npmmirror.com
# 或单次：
npm install --registry=https://registry.npmmirror.com
```

其他地区可用默认 npm 注册源。

### 系统前置要求

- **AKShare One MCP（可选）**：需本地安装 `uv`（Python 包管理器），`uvx` 命令才能运行该 MCP 服务器。安装：`curl -LsSf https://astral.sh/uv/install.sh | sh`
- 其余能力均为 Node/npm 依赖，无额外系统要求。


## 第三方能力与 API (大陆地区和香港地区通用)

### DashScope（通义千问）对话能力：`QWEN_CHAT_MODEL`

- API 配置
  - Model-name=`QWEN_CHAT_MODEL`（按项目可降级 `QWEN_CHAT_MODEL_FALLBACK`）
  - Host=`QWEN_HOST`
  - API-Key=`QWEN_API_KEY`
  - Base_URL=`QWEN_BASE_URL`
  - Workspace=`QWEN_WORKSPACE` · Region=`QWEN_REGION`
- 技术要点
  - 端点：经 `openai` SDK 调用 OpenAI 兼容 `/chat/completions`
  - 参数：在 OpenAI 兼容面使用 `max_completion_tokens`（**不要**用 `max_tokens`）
  - 用途：LLM 对话、JSON 生成、嵌入
  - **中国大陆 / 香港优先选用**
  - 失败时：向调用方返回错误，供组件展示错误并重试——禁止静默空成功
  - 产品行为：见项目需求文档

### DashScope（通义千问）图像能力：`QWEN_IMAGE_MODEL`

- API 配置
  - Model-name=`QWEN_IMAGE_MODEL`
  - Host=`QWEN_HOST`
  - API-Key=`QWEN_API_KEY`
  - Base_URL（原生）=`QWEN_NATIVE_BASE_URL`
- 技术要点
  - 端点：原生 `…/services/aigc/multimodal-generation/generation`（不要假设与 OpenAI `/images` 一一对应）
  - 参数：仅使用 `QWEN_IMAGE_MODEL` **API 文档已记载**的 size、quality、参考图字段——禁止自造尺寸或画质档位
  - 用途：文生图 / 图像编辑
  - **性能调优（组件目标 ≤20s；软提示约 10s——见「性能」）：**
    - 使用任务可接受的**最低**文档尺寸与画质——不要为「更好效果」擅自升高；不支持的参数会失败或浪费时间
    - **每个逻辑请求一张图**（`n: 1` 或厂商等价参数）；禁止对同一输出并行重复调用
    - **在服务端缩小参考图**后再做多模态 / 编辑（例如长边约 1024px，合理 JPEG/WebP），以降低上传与上游处理耗时
    - **短而结构化的提示词**——只写任务摘要；避免重复界面上已展示的长上下文
    - 文本已可用时，**不要**把依赖的文本 LLM 与图像生成串在同一关键路径上——先返回文本，再尽快返回图像
    - 服务端路由：复用单一 HTTP 客户端；出站 **HTTP 超时约 25s**（高于 20s 目标的缓冲）——**不要**另做一套 **10s 硬中止**，在 UI 仍显示进度时把请求掐掉
    - 上游较慢时：保持请求直至超时/错误；客户端约 10s 后提示「还需要更多时间」（见「性能」）——禁止把服务端切断映射成「Timeout」字样
    - 重试：每条尝试路径最多 **一次** 由用户显式触发的重试；禁止在 20s 内自动循环重试
  - 失败时：向调用方返回错误——禁止静默空图
  - 产品行为：见项目需求文档

### 高德地图（AMap）能力：高德开放平台

- API 配置
  - Platform=`AMap (Gaode / 高德地图)`
  - Host=`AMAP_HOST` / JS API，见 `AMAP_KEY_MGMT_SITE`
  - API-Key=`AMAP_API_KEY`
  - Base_URL=`AMAP_BASE_URL`
- 技术要点
  - **中国本地图**——大陆应用默认优先于直连 Google
  - 能力：地理编码、地点搜索、周边 POI、路线、JS 地图；亦用于地点/餐厅等信息补充（本技术栈中替代 Tripadvisor）
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

### Google Maps MCP for China 能力：Cloudflare Worker MCP

- API 配置
  - Service=`Google Maps MCP for China`
  - Host=`GMAPS_MCP_HOST`
  - API-Key=无
  - Bearer=`GMAPS_MCP_BEARER`
  - Base_URL=`GMAPS_MCP_URL`
- 技术要点
  - 传输：Streamable HTTP MCP 代理，在难以直连 Google 时（如中国大陆）使用 Google Maps 工具
  - 仍需 Google 数据、但不做 Google 客户端/服务端直连时使用
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

### Google Maps Platform 能力：Maps Platform（香港可用）

> 仅面向香港（或可直连 Google 的地区）。中国大陆项目请使用高德地图（AMap）或 Google Maps MCP for China。

- API 配置
  - Platform=`Google Maps Platform`
  - Host=`GOOGLE_MAPS_HOST`
  - API-Key=`GOOGLE_MAPS_API_KEY`
  - Project=`GOOGLE_MAPS_PROJECT`
  - Base_URL=`GOOGLE_MAPS_BASE_URL`
- 技术要点
  - API：Places 周边搜索、地理编码（逆 / 正）、Directions、Time Zone、Maps JavaScript、Embed
  - **不要**增加 `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY`——仅使用单一 `GOOGLE_MAPS_API_KEY`（仅服务端 Embed/配置；地点照片需要时经服务端代理；禁止把 Maps key 放进客户端 `img` URL）
  - 中国大陆可能无法直连 Google——见地区限制表及 Maps MCP / 高德
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

### Tripadvisor Terra 能力：Terra API（香港和大陆均可用；大陆数据量少，建议用高德）

> 大陆地区数据量少——面向大陆的项目不要以 Tripadvisor 作为主要数据源，建议使用高德。

- API 配置
  - Service=`Tripadvisor Terra`
  - Host=`TRIPADVISOR_HOST`
  - API-Key=`TRIPADVISOR_API_KEY`（请求头 `X-API-Key`）
  - Base_URL=`TRIPADVISOR_BASE_URL`
- 技术要点
  - 端点：`GET /locations/nearby` · `GET /locations/{id}` · `GET /locations/{id}/reviews`
  - 参数：`lat` + `lon`（不要用 `latLong`）；`radius`；`unit=KM`（大写）
  - **不要**用 Terra 密钥调用 Content API 主机——仅使用 Terra 主机（`TRIPADVISOR_HOST` / `TRIPADVISOR_BASE_URL`）
  - 返回 UI 前，将本地化名称/评论字段展平为普通字符串
  - 失败时：向调用方返回错误，或省略增强信息——禁止静默导致 UI 损坏
  - 产品行为：见项目需求文档

### Open-Meteo 能力：天气预报

- API 配置
  - Service=`Open-Meteo`
  - Host=`OPEN_METEO_HOST`
  - API-Key=无
  - Base_URL=`OPEN_METEO_BASE_URL`
- 技术要点
  - 端点：`GET /forecast`
  - 参数（示例）：`current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m`
  - 用途：通用天气预报（气温、天气代码、湿度、风）
  - 文档：`OPEN_METEO_DOCS`
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

### Finnhub 能力：美股

- API 配置
  - Service=`Finnhub`
  - Host=`FINNHUB_HOST`
  - API-Key=`FINNHUB_API_KEY`（Query 参数 `token`）
  - Base_URL=`FINNHUB_BASE_URL`
- 技术要点
  - 股票行情、公司新闻、市场日历、基础财务 · 免费档 · 文档见 `FINNHUB_DOCS`
  - 示例：`GET /quote?symbol=AAPL&token=…`
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

### AKShare One MCP 能力：中国股票数据（uvx）

- API 配置
  - Service=`AKShare One MCP`
  - Host=`AKSHARE_MCP_HOST`（经 `uvx` 启动的本地 MCP 进程）
  - API-Key=无
  - 命令 / 包=`AKSHARE_MCP_PACKAGE`（用 `uvx` 运行）
- 技术要点
  - **前置要求：** 需安装 `uv`（Python 包管理器），`uvx` 命令才能运行该 MCP 服务器
  - 用途：经 MCP 工具获取中国股票 / 权益市场数据
  - 无需 API Key
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

### Resend 能力：事务性邮件

- API 配置
  - Service=`Resend`（或 SMTP）
  - Host=`RESEND_HOST`（或 `SMTP_URL`）
  - API-Key=`RESEND_API_KEY`（或 SMTP 用 `SMTP_URL`）
  - Base_URL=`RESEND_BASE_URL`
- 技术要点
  - 事务性邮件（如密码重置、引导邮件）
  - 密钥管理：`RESEND_KEY_MGMT_SITE` · `import { Resend } from 'resend'`
  - 失败时：向调用方返回错误——未获提供商确认前不得声称发送成功
  - 产品行为：见项目需求文档

### Pandorium MCP 能力：Streamable HTTP MCP

- API 配置
  - Service=`Pandorium MCP`
  - Host=`PANDORIUM_HOST`
  - API-Key=无
  - Base_URL=`PANDORIUM_MCP_URL`
- 技术要点
  - 传输：Streamable HTTP（JSON-RPC POST，SSE `data:` 响应）
  - 工具（示例）：`tools/list`、`get_planet_positions`、`calculate_natal_chart`、`get_current_transits`
  - 本命盘 / 行运工具需要 `birthData` 及时区——禁止仅用出生日期调用
  - 用途：星座 / 本命盘 / 行星位置与行运
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

### TCGdex 能力：宝可梦 TCG 目录

- API 配置
  - Service=`TCGdex`
  - Host=`TCGDEX_HOST`
  - API-Key=无
  - Base_URL=`TCGDEX_BASE_URL`
- 技术要点
  - 目录、系列、图片、市场价格（TCGPlayer USD、Cardmarket EUR）、短期价格趋势（avg1/7/30）
  - 开源 · 无需密钥 · 文档见 `TCGDEX_DOCS`
  - 冒烟测试 2026-07-24：约 0.6s，卡牌详情含定价 · TCG 交易/价值分析 MVP 优先
  - 失败时：向调用方返回错误——禁止静默空成功
  - 产品行为：见项目需求文档

## 环境变量

将密钥复制到 `.env.local`（已 gitignore；勿提交）。不要在源码中硬编码。规格仅列出**环境变量代码**——本地自行填值。权威清单见 [`keys-cn.md`](./keys-cn.md)。

```env
# QWEN DashScope（通义千问）
QWEN_API_KEY=
QWEN_HOST=
QWEN_BASE_URL=
QWEN_NATIVE_BASE_URL=
QWEN_WORKSPACE=
QWEN_REGION=
QWEN_CHAT_MODEL=
QWEN_CHAT_MODEL_FALLBACK=
QWEN_IMAGE_MODEL=
QWEN_KEY_MGMT_SITE=

# 地图
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_HOST=
GOOGLE_MAPS_PROJECT=
GOOGLE_MAPS_BASE_URL=
GOOGLE_MAP_KEY_MGMT_SITE=
AMAP_API_KEY=
AMAP_HOST=
AMAP_BASE_URL=
AMAP_KEY_MGMT_SITE=
GMAPS_MCP_HOST=
GMAPS_MCP_URL=
GMAPS_MCP_BEARER=

# Tripadvisor 地点
TRIPADVISOR_API_KEY=
TRIPADVISOR_HOST=
TRIPADVISOR_BASE_URL=
TRIPADVISOR_KEY_MGMT_SITE=

# 邮件 / Resend
RESEND_API_KEY=
RESEND_HOST=
RESEND_BASE_URL=
RESEND_KEY_MGMT_SITE=
SMTP_URL=

# 美股
FINNHUB_API_KEY=
FINNHUB_HOST=
FINNHUB_BASE_URL=
FINNHUB_DOCS=

# 天气（无需密钥）
OPEN_METEO_HOST=
OPEN_METEO_BASE_URL=
OPEN_METEO_DOCS=

# 宝可梦 TCG 目录（无需密钥）
TCGDEX_HOST=
TCGDEX_BASE_URL=
TCGDEX_DOCS=

# 星座 MCP
PANDORIUM_HOST=
PANDORIUM_MCP_URL=

# A股 MCP
AKSHARE_MCP_HOST=
AKSHARE_MCP_PACKAGE=

# 无需密钥：Open-Meteo、Pandorium（默认 URL）、TCGdex、AKShare One MCP
```

## 性能

图像生成另有调优约定——见「DashScope（通义千问）图像能力」下的性能调优。

**延迟（所有组件——仅此一套约定）：**

- 软提示：约 **10s** 后展示「还需要更多时间」，同时继续等待（禁止使用「Timeout」字样）
- 目标：每个组件在 **≤20s** 内就绪
- **不要**另做一套冲突的 10s 硬超时，在无用户可见状态时提前放弃组件

**加载策略（所有组件）：**

- 独立组件之间**并行**请求
- 各块完成后即**流式 / 渲染**——不要因最慢的一块卡住整页
- 等待期间展示**按组件**的提示与进度

**降级：**

- 仅在上游失败，或用户可见等待路径结束后组件仍为空时，使用**本地降级**
- **不要**在约 10s 时静默切到降级，而 UI 仍暗示正在实时生成

## 实现陷阱（必须遵守）

实现时对照项目需求文档核对：

| 领域 | 约定 |
| --- | --- |
| **地区 / LLM** | 中国大陆 / 香港 OpenAI 不可用或不稳定——使用 DashScope；不要假设全球统一 LLM 主机 |
| **对话** | OpenAI 兼容对话面使用 `max_completion_tokens`——**不要**用 `max_tokens` |
| **图像（DashScope）** | 原生多模态生成端点——不要假设 OpenAI `/images` 参数一一映射；仅用文档记载的 size/quality；见 DashScope 图像下的性能调优 |
| **地图** | 单一服务端 `GOOGLE_MAPS_API_KEY`——禁止 `NEXT_PUBLIC_*`；中国大陆默认高德，或用 Google Maps MCP；避免从大陆直连 Google 客户端 |
| **Tripadvisor** | 仅使用 Terra 主机；返回 UI 前将本地化字段展平为普通字符串 |
| **AKShare MCP** | 需要本地 `uv` / `uvx`；无 API Key——不要为其伪造 Finnhub 式 token |
| **延迟** | 软提示约 10s · 就绪目标 ≤20s——**仅此一套约定**（见「性能」——所有组件） |
| **密钥** | 仅通过 `.env.local` 环境变量——切勿在规格或仓库中提交真实密钥 |

## 本地化说明（i18n）

本技术栈**不受** always-on 规则中 i18n 默认需求的约束。用户可见文案允许硬编码；项目的本地化策略（支持语言、消息目录等）由该项目需求文档决定。

## 质量门槛与测试

本技术栈每个项目必须满足的最低门槛（依据 `common-test-strategy` 基线，而非其完整默认清单）：

- 业务逻辑必须配单元测试；变更关键路径 **100%** 覆盖
- API / 数据边界路径配集成测试
- 至少一条关键用户旅程用真实浏览器 E2E 覆盖
- 测试使用临时数据目录——绝不用生产数据
- 无数据库：持久化仅用 `data/` 下本地 JSON

**豁免：** 本技术栈不受 `common-test-strategy` 完整默认清单的约束；更严格的项目级测试策略可在各项目需求文档 / 测试策略文档中定义。
