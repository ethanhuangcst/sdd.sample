# 技术规格（高德）
- 本文件是练习项目的通用技术规格。产品行为写在该项目的需求里。
- 后续项目可能同时包含 Web 应用、MCP、RAG 和 Agent。本文件规定这些部分要用到的技术栈、第三方能力和安装方式。
- 项目需求里的安装清单短于本文件时，以需求为准。

## 1.原则
- 目录：`specs/` 只放需求和规格，不放代码。代码按类型分目录，例如 Next.js 用 `web/`，RAG 用 `rag/`，MCP 用 `mcp/`。
- 最小依赖：第 2 节列出可选用的技术，不是每个项目都要装齐。只安装该项目需求点名的包。TypeScript 由 `create-next-app` 选定并安装。
- 本地文件存储：持久化用本地文件，放在项目的 `data/`。不使用数据库，不使用 ORM。
- 第三方调用失败要向界面说明原因。禁止静默空成功、空图、空列表。需求没写重试时，用户可以手动再试一次。禁止自动循环重试。
- 用户可见文案是否使用 i18n，由项目需求决定。未要求时可以直接写一种语言。

## 2.技术栈

下表为主版本（参考日期 2026-10）。具体 patch 以项目的 `package.json` 和 lockfile 为准。用 `create-next-app` 建工程时，使用该命令安装的 TypeScript，不在建工程时改成 7.0。

| 类别 | 技术 | 主版本 |
| --- | --- | --- |
| 框架 | Next.js（App Router） | 16.4 |
| UI / 语言 | React、TypeScript | 19.3、7.0 |
| 样式 | Tailwind CSS | 4.3 |
| 状态 / 表单 | React Query、Zustand、RHF + Zod | 5.x、5.x、7.x |
| 存储 | 本地 JSON（`data/`），无数据库 | — |
| AI SDK | `openai` SDK 只用于对话。图像用 Gemini `generateContent` | 7.x |
| 测试 | Vitest、RTL、Playwright | 5.x、16.x、1.64 |

## 3.能力

### 对话

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6-astra
```

- 提供方是 Quanzil，接口是 OpenAI 兼容的 chat completions。用 `openai` SDK。
- 创建客户端时传入上面的 Key 和 base URL。不要再设置 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL`。
- 用 `max_completion_tokens`，不用 `max_tokens`。`choices[0].message.content` 为空即失败。

### 图像

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

- 与对话共用 `QUANZIL_API_KEY`。`POST` 上面的 URL。请求头 `Authorization: Bearer <QUANZIL_API_KEY>`。
- 正文带 `generationConfig.responseModalities: ["IMAGE"]`。不用 `openai` 的 images 方法。不要设置 `imageConfig.imageSize`。
- 有照片时 `parts` 含文本和 `inlineData`。`mimeType` 用照片本身的类型：`image/png`、`image/jpeg`、`image/webp`、`image/heic`、`image/heif`。先把长边缩到约 1024px。1×1 图片会 400。
- 结果在 `candidates[0].content.parts[].inlineData`。没有 `inlineData` 即失败。读不到照片就报错，禁止改成不带照片的请求。

### 高德

```env
AMAP_API_KEY=
AMAP_HOST=https://restapi.amap.com
```

- 地理编码在 `/v3`，搜索和路线在 `/v5`。不要设一个 `/v3` 的 base 盖住全部接口。
- 坐标是 GCJ-02，顺序是经度,纬度。成功是 HTTP 200、`status` 为 `1`、`infocode` 为 `10000`。
- Key 只放查询参数，只在服务端使用。禁止 `NEXT_PUBLIC_*`。浏览器打开 `https://uri.amap.com/marker?position=经度,纬度`，不加载 JS API。
- 同一个地点只做一次地理编码。发给 Open-Meteo 之前换成 WGS-84，用本地迭代，不安装坐标库。中国境外的坐标不必换算。周边检索仍用原来的 GCJ-02。
- 是否在中国大陆，用返回的 `adcode` 判断。以 71、81、82 开头分别是台湾、香港、澳门，都不是中国大陆。没有结果按解析失败处理。
- 半径和查几次由该项目的需求决定。需求没有写明可以扩大范围时，不要加大半径再查。

### 天气

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

- `GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,weather_code`。坐标为 WGS-84。无需 Key。
- 用已经得到的坐标发一次。不要为了天气再定位一次。
- `weather_code` 是 WMO 代码。展示前译成项目所用语言。禁止把裸代码或英文描述直接给用户。

### 星座 MCP

```env
PANDORIUM_MCP_URL=https://pandorium.app/mcp
```

- 项目需求写明星座服务时才接入。`POST` 该 URL。请求头 `Content-Type: application/json` 和 `Accept: application/json, text/event-stream`。
- 工具名是 `calculate_natal_chart`。参数是 `birthData`：`date`、`time`、`location.latitude`、`location.longitude`、`location.timezone`、`location.city`。无出生时刻时 `time` 用 `12:00`。
- 成功时解析 SSE 的 `data:` 行。正文在 `result.content[0].text`。没有 `content` 即失败。
- 429 不是 JSON-RPC。展示 `retryAfter`。禁止对 429 自动重试。
- 测试用夹具，不出网。冒烟间隔至少 30 秒。

### 宝可梦

```env
TCGDEX_BASE_URL=https://api.tcgdex.net/v2
```

- 项目需求写明卡片目录时才接入。`GET {TCGDEX_BASE_URL}/{lang}/cards/{id}`。无需 Key。
- `lang` 用 `zh-cn`、`zh-tw` 或 `en`。按卡片 id 发一次。不要对同一张卡重复请求。

## 4.安装

- 尚无 `package.json` 时，先按该项目的需求建工程，再安装。已有 lockfile 时用 `npm ci`，不要用 `npm install` 改写 lockfile。
- 中国大陆、香港、澳门：`npm install --registry=https://registry.npmmirror.com`，已有 lockfile 时把命令换成 `npm ci --registry=https://registry.npmmirror.com`。
- 其他地区：`npm install` 或 `npm ci`，不设置镜像。
- 禁止 `npm config set registry`。禁止提交项目 `.npmrc`。
- peer 依赖失败时，给失败的那条命令加上 `--legacy-peer-deps` 再装一次。不要为了通过安装去改 `@types/node`。
- 密钥在 `specs/.secrets`。这个文件由人事先放好。只复制本项目用到的变量名，写入该工程的 `.env.local`。需要的变量不存在或为空则停止。禁止 `NEXT_PUBLIC_*`。回复、日志和浏览器响应里不出现密钥。`.secrets` 和 `.env.local` 都不提交。

## 5.调用顺序

彼此独立的调用同时发出。一块完成就显示这一块。需求写明的先后依赖照需求。失败只说明这一块。不自动重试。

## 6.测试

本节只写工具和跑法。业务用例、夹具和通过标准写在该项目的测试文件里，不写在需求里，也不写在这里。

- 单元和组件用 Vitest。要渲染页面时用 `jsdom` 和 Testing Library。
- 接口和集成用同一套 Vitest，请求打到本机路由。外部地图、对话和图像用夹具，不出网。
- 测试文件点名的浏览器旅程用 Playwright。没有点名就不跑端到端。
- 测试数据放临时目录，不用项目的 `data/`。密钥不进测试代码，不进日志。
- 不在本文件写覆盖率百分比。
