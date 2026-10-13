# 技术规格

本文定义技术栈、第三方能力与通用实现约束。产品功能见各项目需求文档；MyDay 专有约束见第 5 节。英文版见 [`tech-spec.en.md`](./tech-spec.en.md)。

## 1. 第三方能力

只接入项目需要的部分。不接点评服务。失败语义、性能、密钥见第 3 节。

地区写法：`所有地区` 含中国大陆、香港、澳门与其他地区。只服务部分地区的能力会写明范围。

### 对话

#### 地区：所有地区

提供方是 Quanzil（OpenAI 兼容 chat completions）。与下一节图像共用一把 `QUANZIL_API_KEY`。对话用 `openai` SDK。图像的请求形状见下一节。

##### 环境配置

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6-astra
```

##### 请求

2026-10-13 实测：下面的调用返回了非空的 `choices[0].message.content`。

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.QUANZIL_API_KEY,
  baseURL: process.env.QUANZIL_BASE_URL,
});

const completion = await client.chat.completions.create({
  model: process.env.QUANZIL_CHAT_MODEL,
  messages: [{ role: "user", content: "任务摘要" }],
  max_completion_tokens: 1024,
});

const text = completion.choices[0]?.message?.content ?? "";
```

`text` 为空即失败。

##### 关键陷阱

- 创建客户端时传入 `QUANZIL_API_KEY` 与 `QUANZIL_BASE_URL`。禁止再配置 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL`：SDK 会默认读取它们并请求 `api.openai.com`
- 使用 `max_completion_tokens`，不要用 `max_tokens`

---

### 图像生成

#### 地区：所有地区

与对话共用 `QUANZIL_API_KEY`。请求是 Gemini 原生 `generateContent`，不用 `openai` SDK 的 images 方法。

##### 环境配置

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

##### 请求

`POST $QUANZIL_IMAGE_URL`，请求头 `Authorization: Bearer <QUANZIL_API_KEY>` 与 `Content-Type: application/json`。2026-10-13 实测：纯文字请求，以及一张 64×64 JPEG 的 `inlineData`，都返回 HTTP 200，且 `inlineData.mimeType` 为 `image/jpeg`。1×1 JPEG 返回 400 `Unable to process input image`，参考图必须是可解码的照片。

```json
{
  "contents": [{
    "role": "user",
    "parts": [
      { "text": "任务摘要" },
      { "inlineData": { "mimeType": "image/jpeg", "data": "<base64>" } }
    ]
  }],
  "generationConfig": { "responseModalities": ["IMAGE"] }
}
```

无用户照片时，`parts` 只留 `text`。有照片时必须带上 `inlineData`：服务端先缩小（长边约 1024px，JPEG/WebP）再做 base64。读照片失败须报错，禁止删掉 `inlineData` 再请求。

图片在响应的 `candidates[0].content.parts[].inlineData.data`（base64），类型在同级 `mimeType`。没有 `inlineData` 即失败。

##### 关键陷阱

- 必须带 `generationConfig.responseModalities: ["IMAGE"]`。缺了它常只返回文字
- 该模型 id 须出现在这把 Key 的 `GET /v1/models` 中
- 文本已可用时，不要把文本 LLM 与图像串在同一关键路径上
- 提示词短而结构化，只写任务摘要

---

### 地图

#### 地区：中国大陆（高德地图）

地理编码、地点搜索、周边 POI、路线。

##### 环境配置

```env
AMAP_API_KEY=your-web-service-key
AMAP_HOST=https://restapi.amap.com
# GET {AMAP_HOST}/v3/geocode/geo?address={地址}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/text?keywords={关键词}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/around?location={lng},{lat}&radius=3000&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/direction/walking?origin={lng},{lat}&destination={lng},{lat}&key=${AMAP_API_KEY}
```

`location`、`origin`、`destination` 都是 `经度,纬度`。HTTP 200 且 `status` 为 `1`、`infocode` 为 `10000` 才算成功。2026-10-13 四条路径按上面的查询参数实测，`infocode` 均为 `10000`。

##### 关键陷阱

- Key 放在查询参数 `key`，不放请求头。仅服务端使用，禁止 `NEXT_PUBLIC_*`。浏览器地图用 URI H5，不要加载 JS API（MyDay 细节见第 5 节）
- 地理编码在 `/v3`，关键字搜索、周边搜索、步行路线在 `/v5`。禁止设置单一 `AMAP_BASE_URL=/v3` 覆盖全部接口
- 浏览器定位与 Open-Meteo 用 WGS-84；高德 REST 用 GCJ-02，须做坐标转换

#### 地区：中国大陆以外（Google Maps）

含香港、澳门与其他地区。

##### 环境配置

```env
GOOGLE_MAPS_API_KEY=your-server-key
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_DIRECTIONS_URL=https://maps.googleapis.com/maps/api/directions/json
GOOGLE_MAPS_PLACES_SEARCH_NEARBY_URL=https://places.googleapis.com/v1/places:searchNearby
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
GOOGLE_MAPS_PLACES_PHOTO_PREFIX=https://places.googleapis.com/v1
GOOGLE_MAPS_FIELD_MASK=places.displayName,places.formattedAddress,places.location,places.photos
```

##### 关键陷阱

- 单一服务端 `GOOGLE_MAPS_API_KEY`，禁止 `NEXT_PUBLIC_*`。Geocoding、Directions 把 Key 放在查询参数 `key`。Places 与地点照片把 Key 放在请求头 `X-Goog-Api-Key`
- Geocoding：`GET ${GOOGLE_MAPS_GEOCODE_URL}?address={地址}&key=${GOOGLE_MAPS_API_KEY}`。Directions：`GET ${GOOGLE_MAPS_DIRECTIONS_URL}?origin={lat},{lng}&destination={lat},{lng}&mode=walking&key=${GOOGLE_MAPS_API_KEY}`。这里是 `纬度,经度`，与高德的 `经度,纬度` 相反。`status` 为 `OK` 才算成功。2026-10-13 实测两条都返回 `OK`
- Places：`POST` 上面的 search URL，请求头 `Content-Type: application/json`、`X-Goog-Api-Key`、`X-Goog-FieldMask: ${GOOGLE_MAPS_FIELD_MASK}`。`searchNearby` 的正文：

```json
{
  "locationRestriction": {
    "circle": {
      "center": { "latitude": 22.3193, "longitude": 114.1694 },
      "radius": 3000.0
    }
  }
}
```

`searchText` 的正文：

```json
{
  "textQuery": "粤菜",
  "locationBias": {
    "circle": {
      "center": { "latitude": 22.3193, "longitude": 114.1694 },
      "radius": 3000.0
    }
  }
}
```

`radius` 单位是米。少了 `circle` 会 400。2026-10-13 实测这两个正文都返回 HTTP 200，响应里有 `places`。
- 地点照片：`GET ${GOOGLE_MAPS_PLACES_PHOTO_PREFIX}/{photo.name}/media?maxHeightPx=800`，请求头 `X-Goog-Api-Key`。`photo.name` 的值是 `places/{placeId}/photos/{photoId}`。必须带 `maxHeightPx` 或 `maxWidthPx`（1–4800），否则请求被拒绝。默认会重定向到图片，服务端跟随重定向取回字节再交给浏览器。2026-10-13 实测该请求跟随重定向后为 `image/jpeg`。客户端 `img` 地址指向自己的路由，不带 Key
- Geocoding、Directions 与 Places 的 host 不同，禁止把 `https://maps.googleapis.com` 当成统一 base
- Key 的应用限制使用 IP 地址；本机开发可暂不限制。禁止用 HTTP 引用来源限制，服务端请求没有 Referer，会返回 `REQUEST_DENIED`
- 在 Google Cloud 只启用实际调用的 API

---

### npm 镜像

#### 地区：中国大陆、香港和澳门

##### 环境配置

```bash
npm ci --registry=https://registry.npmmirror.com
```

##### 关键陷阱

- 已有 lockfile 时用 `npm ci`，不要用 `npm install --registry=...`。后者会把 `resolved` 改写成镜像地址
- lockfile 里的 `registry.npmjs.org` 表示「当前配置的 registry」。`npm ci --registry=https://registry.npmmirror.com` 从镜像下载，不改 lockfile
- 禁止 `npm config set registry`，禁止提交项目 `.npmrc`
- `npm ci` 遇 `edgesOut` 空指针时改为 `npm ci --registry=https://registry.npmmirror.com --legacy-peer-deps`，不要改 lockfile 里的版本

#### 地区：其他地区

##### 环境配置

```bash
npm ci
```

##### 关键陷阱

- 已有 lockfile 时用 `npm ci`。不要用 `npm install` 去「刷新」依赖
- 禁止 `npm config set registry`，禁止提交项目 `.npmrc`
- 若 `npm config get registry` 不是 `https://registry.npmjs.org/`，用 `npm config delete registry` 清掉用户级设置后再 `npm ci`

---

### 天气

#### 地区：所有地区

##### 环境配置

```env
# GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m&timezone=auto
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

无需 Key。`latitude`、`longitude` 为 WGS-84。2026-10-13 实测该查询返回 `current`，其中含 `temperature_2m`、`relative_humidity_2m`、`weather_code`、`wind_speed_10m`。

##### 关键陷阱

- `weather_code` 为 WMO 代码，展示前须映射为目标语言文案，禁止把裸代码或英文描述直接给用户

---

### 星座 MCP

#### 地区：所有地区

##### 环境配置

```env
# Streamable HTTP MCP，无需 Key
PANDORIUM_MCP_URL=https://pandorium.app/mcp
```

##### 请求

`POST $PANDORIUM_MCP_URL`，请求头 `Content-Type: application/json` 与 `Accept: application/json, text/event-stream`。2026-10-13 实测：下面的 `tools/call` 返回 SSE，`event: message` 后面一行 `data:` 为 JSON-RPC，`result.content` 为文本。`tools/list` 的 `inputSchema` 要求 `birthData`，且 `location` 必填 `latitude`、`longitude`、`timezone`、`city`。

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "calculate_natal_chart",
    "arguments": {
      "birthData": {
        "date": "1990-03-15",
        "time": "12:00",
        "location": {
          "latitude": 31.2304,
          "longitude": 121.4737,
          "timezone": "Asia/Shanghai",
          "city": "上海"
        }
      }
    }
  }
}
```

`get_current_transits` 使用同一个 `birthData`，`params.name` 改为该工具名。成功时解析 SSE 的 `data:` 行：`result.content` 是数组，正文在 `result.content[0].text`（一段 JSON 字符串）。没有 `content` 即失败。无出生时刻时 `time` 固定 `12:00`（MyDay 见第 5 节）。

##### 关键陷阱

- 禁止只传出生日期。`location` 缺 `city` 或 IANA `timezone` 会被 schema 拒绝
- 429 响应体为 `{"success":false,"error":"Too many requests","retryAfter":…}`，不是 JSON-RPC；须展示 `retryAfter`，禁止对 429 自动重试
- 单元 / 集成 / CI 用夹具，禁止出网打 live MCP；冒烟间隔 ≥30 秒
- 与天气并行、同一档案同一日的请求次数，见使用该能力的项目附录（MyDay 在第 5 节）

---

### 宝可梦 TCG 目录

#### 地区：所有地区

##### 环境配置

```env
# GET {TCGDEX_BASE_URL}/{lang}/cards/{id} ，无需 Key
# lang：zh-cn、zh-tw、en。例：/zh-cn/cards/SV7-001
TCGDEX_BASE_URL=https://api.tcgdex.net/v2
```

2026-10-13 实测 `GET /v2/zh-cn/cards/SV7-001` 与 `GET /v2/en/cards/base1-4` 返回 200。路径里的 `zh` 会返回 `language-invalid`。

##### 关键陷阱

- `lang` 用 `zh-cn`、`zh-tw` 或 `en`。只使用文档记载的路径与字段

## 2. 技术栈

下表为主版本（参考日期 2026-10）。具体 patch 以你项目根目录的 `package.json` / lockfile 为准。

| 类别 | 技术 | 主版本 |
| --- | --- | --- |
| 框架 | Next.js（App Router） | 16.4 |
| UI / 语言 | React · TypeScript | 19.3 · 7.0（native port；本地先确认编辑器与构建通过） |
| 样式 | Tailwind CSS | 4.3 |
| 状态 / 表单 | React Query · Zustand · RHF + Zod + `@hookform/resolvers` | 5.x · 5.x · 7.x + 4.x + 5.x |
| 存储 | 本地 JSON（`data/`）— 无数据库 | — |
| AI SDK | `openai` SDK 仅用于对话（Quanzil chat completions）。图像为 Gemini `generateContent`，见第 1 节 | 7.x（6→7 有 breaking changes，升级时过官方迁移指南） |
| 测试 | Vitest · RTL · Playwright | 5.x · 16.x · 1.64 |

规则：尽量少依赖；优先平台与标准库；**禁止数据库**（除非项目规格另有说明，否则仅用 `data/` 下本地 JSON）。npm 镜像见第 1 节。

## 3. 通用契约（所有项目、所有能力）

### 3.1 失败语义

- 第三方调用失败：向调用方返回错误，供组件展示并允许重试——禁止静默空成功、空图、空列表。
- 降级仅在上游失败，或用户可见等待路径结束后组件仍为空时出现。禁止在等待期间静默切降级。

### 3.2 性能（所有组件）

- 软提示：约 **10s** 后展示「还需要更多时间」，同时继续等待——禁止把「Timeout」字样给用户。
- 目标：每个组件 **≤20s** 就绪。
- 服务端出站超时约 **25s**——不要另做一套 10s 硬中止掐断仍在进行的请求。
- 重试：每条尝试路径最多一次由用户显式触发；禁止 20s 内自动循环重试。
- 加载：独立组件并行请求；各块完成即渲染；等待期间展示按组件的提示与进度。

### 3.3 密钥

- 运行时密钥仅写 `.env.local`（已 gitignore，勿提交）。下文只列环境变量名称。
- 禁止 `NEXT_PUBLIC_*` 暴露地图或 LLM 密钥。模型 id 用环境变量锁死，不依赖提供方默认模型。

### 3.4 质量门槛

- 业务逻辑配单元测试；变更关键路径 100% 覆盖
- API / 数据边界配集成测试（夹具，默认不出网）
- 至少一条关键用户旅程用真实浏览器 E2E 覆盖
- 测试用临时数据目录
- 无数据库：持久化仅用 `data/` 下本地 JSON

### 3.5 文案与 i18n

用户可见文案是否使用 i18n 由该项目需求文档决定；未要求 i18n 时可直接硬编码。

### 3.6 星座与天气

限流、`birthData` 与测试约束见第 1 节「星座 MCP」。与天气并行、同一档案同一日的请求次数见第 5 节。

## 4. 环境变量汇总

各能力已在第 1 节给出 `.env.local` 样例。项目只启用用到的变量；可选测试目录：

```env
DATA_DIR=
```

## 5. 项目附录：MyDay

产品功能见 [`../projects/myday/req.zh.md`](../projects/myday/req.zh.md)。以下为 MyDay 的实现约束。对话 / 图像 / 地图见第 1 节（对话与图像均经 Quanzil：对话 `gpt-6-astra`，图像 `gemini-3.1-flash-lite-image`；大陆高德，大陆以外 Google Maps）。

### 依赖边界

不新增：Prisma / 数据库、MCP SDK、高德 JS API、`NEXT_PUBLIC_*` 密钥。图像缩小使用 Next 已带的 `sharp`。坐标转换、WMO→中文、太阳星座、MCP JSON-RPC 用标准库 / `fetch` 自写。

### 存储与路由

| 路径 | 职责 |
| --- | --- |
| `data/profiles.json` | 档案数组（id、姓名、性别、出生日期、身高、体重、职业、`photoPath` 文件名或空） |
| `data/uploads/` | 用户照片与下载后的穿搭图 |
| 测试 | `DATA_DIR` 指向临时目录；默认 `data/` |

`photoPath` 只存文件名，从 `data/uploads/` 读取（`uploadsDir()`，不要 `cwd + photoPath`）。读失败须报错。有档案照片时必须把该文件作为 `generateContent` 的 inline image，禁止静默改走不带图片的请求。

服务端 Route Handler 不把密钥返回客户端。今日计划分块（天气、星座、穿搭文字、穿搭图、餐厅列表）独立、可并行，失败只影响本块。

### 高德（大陆）

- REST 只走服务端：地理编码 `GET /v3/geocode/geo`；周边搜索 `GET /v5/place/around`，`location=lng,lat`（经度在前）、`radius=3000`、`sortrule=distance`
- 浏览器地图用 URI H5，不用 JS API：标注 `https://uri.amap.com/marker?position={lng},{lat}&name={name}&src=myday&coordinate=gaode`；导航 `https://uri.amap.com/navigation?to={lng},{lat},{name}&mode=walk&src=myday&callnative=1`。浮层用 iframe 或跳转；Escape / 点击背景关闭
- 坐标系：浏览器定位与 Open-Meteo 用 WGS-84；高德 REST / URI 用 GCJ-02。浏览器定位保留 WGS-84 查天气，转 GCJ-02 后再调高德。手动城市的地理编码结果已是 GCJ-02，搜餐厅不要再转；查天气时 GCJ-02 → WGS-84
- 坐标约 `35.86, 104.20`（容差约 0.05°）或 `accuracy > 8000m` 视为无效定位，禁止进入今日计划
- 周边搜固定 3 公里，有结果即停。检索阶梯：`keywords` + `types=050000` → 仅 `keywords` → 仅 `types=050000`
- 菜系关键词：日式→`日本料理`；西班牙式→`西班牙菜`；意式→`意大利菜`；快餐→`快餐`；粤菜→`粤菜`
- `infocode != 10000` 或空列表：返回错误或「换菜系 / 改地点」文案

香港和澳门 MyDay 使用 Google Maps（见第 1 节「地图」），地点照片经服务端代理。

### 天气与星座

- Open-Meteo：`latitude` `longitude`（WGS-84）+ `timezone=auto`。`weather_code` 必须映射为中文后再展示
- Pandorium 用 `fetch` 解析 SSE，不加 MCP SDK。`birthData`：

```json
{
  "date": "1990-03-15",
  "time": "12:00",
  "location": {
    "latitude": 31.2304,
    "longitude": 121.4737,
    "timezone": "Asia/Shanghai",
    "city": "上海"
  }
}
```

- 无出生时刻时 `time` 固定 `12:00`。无效定位时不调用 MCP。太阳星座与年龄本地计算
- 天气与星座请求并行；禁止把 `weather` 放进星座请求的 `useEffect` 依赖
- 同一档案、同一日，本命盘与行运各最多请求一次

### 测试要点

关键路径单元覆盖须含：无效定位判定、WGS-84↔GCJ-02、WMO→中文、太阳星座、菜系关键词、有照片必须带参考图。E2E：档案 → 规划 → 今日计划分块渲染。

### i18n

本项目界面仅中文，文案可硬编码。
