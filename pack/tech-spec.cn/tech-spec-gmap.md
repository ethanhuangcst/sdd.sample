# 技术规格（Google Maps）

- 本文件是练习项目的通用技术规格。产品行为写在该项目的需求里。
- 后续项目可能同时包含 Web 应用、MCP、RAG 和 Agent。本文件规定这些部分要用到的技术栈、第三方能力和安装方式。
- 项目需求里的安装清单短于本文件时，以需求为准。

## 1.原则

- 本文件是练习项目的通用技术规格。产品行为写在该项目的需求里。
- 后续项目可能同时包含 Web 应用、MCP、RAG 和 Agent。本文件规定这些部分要用到的技术栈、第三方能力和安装方式。项目需求没有写的能力，不接入。
- 英文版见 [`../tech-spec.en/tech-spec.md`](../tech-spec.en/tech-spec.md)。
- 第三方调用失败要向界面说明原因。禁止静默空成功、空图、空列表。需求没写重试时，用户可以手动再试一次。禁止自动循环重试。
- 项目需求里的安装清单或测试章短于本文件时，以需求为准。
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
- `gpt-6-astra` 会先把推理 token 计入 `max_completion_tokens`，再写 `choices[0].message.content`。需求提示词里的字数和字符数不设置这个上限。状态总结、行程、穿搭这三次对话都把 `max_completion_tokens` 设为 1200。`choices[0].message.content` 为空即失败，包括 `finish_reason` 为 `length`、推理用完上限、正文为空的情况。

### 图像

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

- 与对话共用 `QUANZIL_API_KEY`。`POST` 上面的 URL。请求头 `Authorization: Bearer <QUANZIL_API_KEY>`。
- 正文带 `generationConfig.responseModalities: ["IMAGE"]`。不用 `openai` 的 images 方法。不要设置 `imageConfig.imageSize`。
- 有照片时 `parts` 含文本和 `inlineData`。`mimeType` 用照片本身的类型：`image/png`、`image/jpeg`、`image/webp`、`image/heic`、`image/heif`。照片是小图：上传由用户控制，长边约 2048px 以内、文件 2MB 以内，应用不做缩图。生图请求加 75 秒超时，超时按该区域失败处理，不重试。1×1 图片会 400。
- 结果在 `candidates[0].content.parts[].inlineData`。没有 `inlineData` 即失败。读不到照片就报错，禁止改成不带照片的请求。

### Google Maps

```env
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_DIRECTIONS_URL=https://maps.googleapis.com/maps/api/directions/json
GOOGLE_MAPS_PLACES_SEARCH_NEARBY_URL=https://places.googleapis.com/v1/places:searchNearby
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
GOOGLE_MAPS_PLACES_PHOTO_PREFIX=https://places.googleapis.com/v1
GOOGLE_MAPS_FIELD_MASK=places.displayName,places.formattedAddress,places.location,places.photos
```

- 地理编码：`GET ${GOOGLE_MAPS_GEOCODE_URL}?address={地址}&key=`。`status` 为 `OK` 才算成功。坐标是 WGS-84，顺序是纬度,经度。
- 路线：`GET ${GOOGLE_MAPS_DIRECTIONS_URL}?origin={纬度},{经度}&destination={纬度},{经度}&mode=walking&key=`。
- 周边：`POST` searchNearby。正文是 `locationRestriction.circle`。少了 `circle` 会 400。
- 文字检索：`POST` searchText。正文是 `textQuery` 加 `locationBias.circle`。少了 `circle` 会 400。
- 地点照片：`GET ${GOOGLE_MAPS_PLACES_PHOTO_PREFIX}/{photo.name}/media?maxHeightPx=800`，请求头 `X-Goog-Api-Key`。必须带 `maxHeightPx` 或 `maxWidthPx`。
- Places 请求头还有 `Content-Type: application/json` 和 `X-Goog-FieldMask`。Geocoding 与 Places 的 host 不同，禁止把 `https://maps.googleapis.com` 当成统一 base。
- Key 只在服务端使用。禁止 `NEXT_PUBLIC_*`。禁止用 HTTP 引用来源限制这把 Key。
- 浏览器打开 `https://www.google.com/maps/dir/?api=1&destination=纬度,经度`。不加载 JS API。
- 同一个地点只做一次地理编码。
- 是否在中国大陆，看结果里 `country` 的短码。`CN` 是中国大陆。`HK`、`MO`、`TW` 和其他国家不是。
- 半径由该项目的需求决定。需求没有写明可以扩大范围时，不要加大半径再查。

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
