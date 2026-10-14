# 技术规格

本文定义技术栈、第三方能力与通用实现约束。产品功能见各项目 `specs/req-spec.md`（种子来自 `pack/projects/<locale>/req-spec.md`）。英文版见 [`../tech-spec.en/tech-spec.md`](../tech-spec.en/tech-spec.md)。

## 1. 第三方能力

只接入项目需要的部分。不接点评服务。失败语义、性能、密钥见第 3 节。

地区写法：`所有地区` 含中国大陆、香港、澳门与其他地区。只服务部分地区的能力会写明范围。

### 对话 LLM

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

##### 性能要求

- 一次短对话。同一次用户等待里，不要把对话排在图像前面。

##### 关键陷阱

- 创建客户端时传入 `QUANZIL_API_KEY` 与 `QUANZIL_BASE_URL`。禁止再配置 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL`：SDK 会默认读取它们并请求 `api.openai.com`
- 使用 `max_completion_tokens`，不要用 `max_tokens`

---

### 图像生成 LLM

#### 地区：所有地区

与对话共用 `QUANZIL_API_KEY`。请求是 Gemini 原生 `generateContent`，不用 `openai` SDK 的 images 方法。

##### 环境配置

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

##### 请求

`POST $QUANZIL_IMAGE_URL`，请求头 `Authorization: Bearer <QUANZIL_API_KEY>` 与 `Content-Type: application/json`。参考图的 `mimeType` 用照片本身的类型：`image/png`、`image/jpeg`、`image/webp`、`image/heic`、`image/heif`。2026-10-13 实测：纯文字请求，以及一张 64×64 JPEG 的 `inlineData`，都返回 HTTP 200，且响应里的 `inlineData.mimeType` 为 `image/jpeg`。1×1 JPEG 返回 400 `Unable to process input image`，参考图必须是可解码的照片。

```json
{
  "contents": [{
    "role": "user",
    "parts": [
      { "text": "任务摘要" },
      { "inlineData": { "mimeType": "image/png", "data": "<base64>" } }
    ]
  }],
  "generationConfig": { "responseModalities": ["IMAGE"] }
}
```

无用户照片时，`parts` 只留 `text`。有照片时必须带上 `inlineData`：服务端先缩小（长边约 1024px）再做 base64。`mimeType` 与缩小后的字节一致，保持在上面五种之内，不要一律改成 JPEG。读照片失败须报错，禁止删掉 `inlineData` 再请求。

图片在响应的 `candidates[0].content.parts[].inlineData.data`（base64），类型在同级 `mimeType`。没有 `inlineData` 即失败。

##### 性能要求

- 参考图长边缩到约 1024px 后立刻发出，保留照片原来的类型。不要为了加速再缩得更小，也不要为了调用改成 JPEG。输出仍是约 1024×1024，再缩小参考图不缩短生成时间。
- 不要设置 `imageConfig.imageSize`。设为 `512` 会返回 400。
- 照片和短提示词齐了就发出。同一次等待里，不要在图像前面再串一次对话。
- 图像单独等待。不要用地图或天气的短等待去切断仍在生成的图。

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

##### 性能要求

- 同一个地点只做一次地理编码。天气和餐厅用当前地点的坐标。星盘用出生城市的坐标。两处不是同一个地点时，各编码一次。

##### 关键陷阱

- Key 放在查询参数 `key`，不放请求头。仅服务端使用，禁止 `NEXT_PUBLIC_*`。浏览器地图用 `https://uri.amap.com/marker?position=经度,纬度`，不要加载 JS API
- 地理编码在 `/v3`，关键字搜索、周边搜索、步行路线在 `/v5`。禁止设置单一 `AMAP_BASE_URL=/v3` 覆盖全部接口
- 浏览器定位与 Open-Meteo 用 WGS-84；高德 REST 用 GCJ-02。WGS-84 与 GCJ-02 自行换算，不新增依赖。判断是否在中国大陆、写入星盘和天气之前，先把高德坐标转成 WGS-84。查餐厅仍用原来的 GCJ-02。手输城市的高德结果已是 GCJ-02
- 是否在中国大陆，用地理编码返回的行政区判断，不写经纬度方框。浏览器坐标先走 `GET /v3/geocode/regeo?location={经度},{纬度}`。`adcode` 以 71、81、82 开头分别是台湾、香港、澳门，都不是中国大陆。正向地理编码没有结果时，按解析失败处理

##### 周边餐饮检索

项目要按菜系找附近餐厅时用这一节。半径是否扩大由该项目的需求决定。

- 用 `GET /v5/place/around`，不用 `/v5/place/text`
- `location` 为 GCJ-02，顺序是经度,纬度。`radius` 的单位是米
- 餐饮类型为 `types=050000`
- 检索词用用户输入的文字
- 有结果就停：先检索词加 `types=050000`，再只检索词，最后只 `types=050000`
- 需求没有写明可以扩大范围时，不要加大 `radius` 再查

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

##### 性能要求

- 同一地点只做一次地理编码。后面的地点检索复用这次坐标，不要再打一次。

##### 关键陷阱

- 单一服务端 `GOOGLE_MAPS_API_KEY`，禁止 `NEXT_PUBLIC_*`。Geocoding、Directions 把 Key 放在查询参数 `key`。Places 与地点照片把 Key 放在请求头 `X-Goog-Api-Key`。浏览器地图用 `https://www.google.com/maps/dir/?api=1&destination=纬度,经度`，不要加载 JS API
- 是否在中国大陆，看地理编码结果里 `country` 的短码。`CN` 是中国大陆。`HK`、`MO`、`TW` 和其他国家不是。浏览器坐标先做一次逆地理编码。不写经纬度方框
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
# 还没有 lockfile 时，只用来生成它
npm install --registry=https://registry.npmmirror.com
# 已有 lockfile 时
npm ci --registry=https://registry.npmmirror.com
```

##### 关键陷阱

- 已有 lockfile 时只用 `npm ci`。不要用 `npm install` 改写它，否则 `resolved` 会变成镜像地址
- lockfile 里的 `registry.npmjs.org` 表示「当前配置的 registry」。`npm ci --registry=https://registry.npmmirror.com` 从镜像下载，不改 lockfile
- 禁止 `npm config set registry`，禁止提交项目 `.npmrc`
- `npm ci` 遇 `edgesOut` 空指针时改为 `npm ci --registry=https://registry.npmmirror.com --legacy-peer-deps`，不要改 lockfile 里的版本

#### 地区：其他地区

##### 环境配置

```bash
# 还没有 lockfile 时，只用来生成它
npm install
# 已有 lockfile 时
npm ci
```

##### 关键陷阱

- 已有 lockfile 时只用 `npm ci`。不要用 `npm install` 刷新依赖
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

##### 性能要求

- 使用已经得到的 WGS-84 坐标发一次预报。不要为这次预报重新定位。

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

`get_current_transits` 使用同一个 `birthData`，`params.name` 改为该工具名。成功时解析 SSE 的 `data:` 行：`result.content` 是数组，正文在 `result.content[0].text`（一段 JSON 字符串）。没有 `content` 即失败。无出生时刻时 `time` 固定 `12:00`。

##### 性能要求

- 使用已经解析的出生城市坐标发一次 `calculate_natal_chart`。没有出生时刻时，`time` 直接用 `12:00`。

##### 关键陷阱

- 禁止只传出生日期。`location` 缺 `city` 或 IANA `timezone` 会被 schema 拒绝
- 429 响应体为 `{"success":false,"error":"Too many requests","retryAfter":…}`，不是 JSON-RPC；须展示 `retryAfter`，禁止对 429 自动重试
- 单元 / 集成 / CI 用夹具，禁止出网打 live MCP；冒烟间隔 ≥30 秒
- 与天气是否并行、同一档案同一日请求几次，由该项目的需求文档规定

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

##### 性能要求

- 按卡片 id 发一次 GET。不要对同一张卡重复请求。

##### 关键陷阱

- `lang` 用 `zh-cn`、`zh-tw` 或 `en`。只使用文档记载的路径与字段

## 设计原则

- 依赖与持久化见第 3.3 节。
- 密钥见第 3.4 节。密钥明文放在 `specs/.secrets`。运行时只读 `.env.local`。这两个文件都不提交。回复、日志和客户端响应里不出现密钥。

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

依赖与持久化见第 3.3 节。npm 镜像见第 1 节。表里的 TypeScript 7.0 是目标版本。用 `create-next-app` 建工程时，用该命令安装的 TypeScript，不在建工程时改成 7.0。安装清单里有的包，版本按本表。清单没有的包不安装，见第 3.3 节。

## 3. 通用契约（所有项目、所有能力）

### 3.1 失败语义

- 第三方调用失败：向调用方返回错误，供界面说明原因。禁止静默空成功、空图、空列表。
- 这一次失败后是否让用户再试一次，由该项目需求决定。需求没写时，用户可以手动再试一次。禁止自动循环重试。
- 降级仅在上游失败，或用户可见等待结束后仍为空时出现。禁止在等待期间静默切降级。

### 3.2 性能（所有组件）

- 彼此独立的调用同时发出。
- 一块完成后就显示这一块。需求写明先取数、下一页再显示时，按需求。
- 某次调用没有结果时，按第 3.1 节说明原因并结束这次等待。怎么做到写在该能力的「性能要求」里。

### 3.3 最小依赖

- 不引入项目用不到的框架。优先平台与标准库。
- 项目需求里的安装清单短于第 2 节的表时，以需求为准。
- 持久化只用本地文件。档案与上传文件放在 `data/` 下。禁止数据库，禁止 ORM。

### 3.4 密钥

- 运行时密钥仅写 `.env.local`（已 gitignore，勿提交）。下文只列环境变量名称。
- 禁止 `NEXT_PUBLIC_*` 暴露地图或 LLM 密钥。模型 id 用环境变量锁死，不依赖提供方默认模型。

### 3.5 质量门槛

- 业务逻辑配单元测试；变更关键路径 100% 覆盖
- API / 数据边界配集成测试（夹具，默认不出网）
- 至少一条关键用户旅程用真实浏览器 E2E 覆盖
- 测试用临时数据目录
- 项目需求里有测试章，且短于本节时，以该章为准。

### 3.6 文案与 i18n

用户可见文案是否使用 i18n 由该项目需求文档决定；未要求 i18n 时可直接硬编码。

### 3.7 星座与天气

限流、`birthData` 与测试约束见第 1 节「星座 MCP」。与天气是否并行、同一日请求几次，由该项目的需求文档规定。

## 4. 环境变量汇总

各能力已在第 1 节给出 `.env.local` 样例。项目只启用用到的变量；可选测试目录：

```env
DATA_DIR=
```
