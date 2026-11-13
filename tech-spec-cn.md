# 技术规格

- 本文件列出全部支持的能力（能力全集）。产品行为写在各项目的需求文件里。
- 使用时先按该项目需求删除不需要的能力小节，只保留用到的部分，避免无关能力进入上下文。
- 后续项目可能同时包含 Web 应用、MCP、RAG 和 Agent。本文件规定这些部分要用到的技术栈、第三方能力和安装方式。项目需求没有写的能力，不接入。
- 第三方调用失败要向界面说明原因。禁止静默空成功、空图、空列表。需求没写重试时，用户可以手动再试一次。禁止自动循环重试。
- 降级只在上游失败、或用户可见等待结束后仍为空时出现；禁止在等待期间静默切降级。
- 项目需求里的安装清单或测试章短于本文件时，以需求为准。
- 用户可见文案是否使用 i18n，由项目需求决定。未要求时可以直接写一种语言。

## 1.原则

- 项目文档在仓库根的 `<项目>/` 目录。代码按类型分目录：Next.js 用 `web/`，RAG 用 `rag/`，MCP 用 `mcp/`。
- 最小依赖：第 2 节列出可选用的技术，不是每个项目都要装齐。只安装该项目需求点名的包。TypeScript 由 `create-next-app` 选定并安装。
- 本地文件存储：持久化用本地文件，放在项目的 `data/`。不使用数据库，不使用 ORM。
- 同一地区有多家供应商时（如地图的高德与百度），用项目需求指定的那一家。

## 2.技术栈

下表为主版本（参考日期 2026-10）。具体 patch 以项目的 `package.json` 和 lockfile 为准。TypeScript 7.0 是目标版本；用 `create-next-app` 建工程时，使用该命令安装的 TypeScript，不在建工程时改成 7.0。

| 类别 | 技术 | 主版本 |
| --- | --- | --- |
| 框架 | Next.js（App Router） | 16.4 |
| UI / 语言 | React、TypeScript | 19.3、7.0 |
| 样式 | Tailwind CSS | 4.3 |
| 状态 / 表单 | React Query、Zustand、RHF + Zod + `@hookform/resolvers` | 5.x、5.x、7.x + 4.x |
| 存储 | 本地 JSON（`data/`），无数据库 | — |
| AI SDK | `openai` SDK 只用于对话；图像用 Gemini `generateContent`（见第 3 节） | 7.x（6→7 有 breaking changes，升级时过官方迁移指南） |
| 测试 | Vitest、RTL、Playwright | 5.x、16.x、1.64 |

## 3.能力

每项能力按四段记录：环境配置、请求（含实测记录）、性能要求、关键陷阱。实测日期写在结论旁。

使用时按该项目需求删除本节不需要的能力小节，避免无关能力进入上下文。

### 对话 LLM

#### 环境配置

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6.1-sol
```

#### 请求

提供方是 Quanzil，OpenAI 兼容 chat completions，用 `openai` SDK。2026-10-13 实测：下面的调用返回非空 `choices[0].message.content`。

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

`text` 为空即失败。`gpt-6.1-sol` 会先把推理 token 计入 `max_completion_tokens`，再写正文；需求提示词里的字数和字符数不设置这个上限。需求为单次文字输出时，把 `max_completion_tokens` 设为 1200。正文为空即失败，包括 `finish_reason` 为 `length`、推理用完上限、正文为空的情况。

#### 性能要求

- 一次短对话。同一次用户等待里，不要把对话排在图像前面。

#### 关键陷阱

- 创建客户端时传入 `QUANZIL_API_KEY` 与 `QUANZIL_BASE_URL`。禁止再配置 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL`：SDK 会默认读取它们并请求 `api.openai.com`。
- 使用 `max_completion_tokens`，不要用 `max_tokens`。

### 图像生成 LLM

#### 环境配置

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

与对话共用 `QUANZIL_API_KEY`。请求是 Gemini 原生 `generateContent`，不用 `openai` SDK 的 images 方法。

#### 请求

`POST $QUANZIL_IMAGE_URL`，请求头 `Authorization: Bearer <QUANZIL_API_KEY>` 与 `Content-Type: application/json`：

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

- 参考图的 `mimeType` 用照片本身的类型：`image/png`、`image/jpeg`、`image/webp`、`image/heic`、`image/heif`，不要一律改成 JPEG。
- 照片由用户控制上传：长边约 2048px 以内、文件 2MB 以内，应用不做缩图。
- 无用户照片时 `parts` 只留 `text`；有照片必须带 `inlineData`，读不到照片就报错，禁止删掉照片再请求。
- 2026-10-13 实测：纯文字请求与一张 64×64 JPEG 的 `inlineData` 都返回 HTTP 200，响应 `inlineData.mimeType` 为 `image/jpeg`；1×1 JPEG 返回 400 `Unable to process input image`，参考图必须是可解码的照片。
- 结果在 `candidates[0].content.parts[].inlineData`（base64），类型在同级 `mimeType`。没有 `inlineData` 即失败。
- 生图请求加 75 秒超时，超时按该区域失败处理，不重试。

#### 性能要求

- 参考图和短提示词齐了就发出。同一次等待里，不要在图像前面再串一次对话。
- 图像单独等待。不要用地图或天气的短等待去切断仍在生成的图。
- 输出约 1024×1024；再缩小参考图不缩短生成时间。

#### 关键陷阱

- 必须带 `generationConfig.responseModalities: ["IMAGE"]`，缺了常只返回文字。
- 该模型 id 须出现在这把 Key 的 `GET /v1/models` 中。
- 不要设置 `imageConfig.imageSize`，设为 `512` 会返回 400。
- 文本已可用时，不要把文本 LLM 与图像串在同一关键路径上。


### 高德（AMap）— 中国大陆

#### 环境配置

```env
AMAP_API_KEY=
AMAP_HOST=https://restapi.amap.com
# GET {AMAP_HOST}/v3/geocode/geo?address={地址}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/text?keywords={关键词}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/around?location={经度},{纬度}&radius=3000&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/direction/walking?origin={经度},{纬度}&destination={经度},{纬度}&key=${AMAP_API_KEY}
```

#### 请求

- `location`、`origin`、`destination` 都是经度,纬度。HTTP 200 且 `status` 为 `1`、`infocode` 为 `10000` 才算成功。2026-10-13 四条路径按上面的查询参数实测，`infocode` 均为 `10000`。
- 地理编码在 `/v3`，关键字搜索、周边搜索、步行路线在 `/v5`；不要设一个 `/v3` 的 base 盖住全部接口。
- 周边餐饮检索（项目要按菜系找附近餐厅时用）：
  - 用 `GET /v5/place/around`，不用 `/v5/place/text`。`location` 为 GCJ-02 经度,纬度，`radius` 单位是米，餐饮类型 `types=050000`，检索词用用户输入的文字。
  - 有结果就停：先检索词加 `types=050000`，再只检索词，最后只 `types=050000`。

#### 性能要求

- 同一个地点只做一次地理编码。天气和餐厅用当前地点的坐标；出生城市是另一地点时各编码一次。

#### 关键陷阱

- Key 放查询参数 `key`，不放请求头。浏览器打开 `https://uri.amap.com/marker?position=经度,纬度`。
- 浏览器定位与 Open-Meteo 用 WGS-84；高德 REST 用 GCJ-02。发给 Open-Meteo 前用公开迭代在本地换算，不安装坐标库；中国境外的坐标不必换算；周边检索保持 GCJ-02。
- 是否在中国大陆用返回的 `adcode` 判断：71、81、82 开头分别是台湾、香港、澳门，都不是中国大陆；正向地理编码没有结果按解析失败处理。浏览器定位坐标先走 `GET /v3/geocode/regeo?location={经度},{纬度}`，再用 `adcode` 判断；不写经纬度方框。

### 百度（Baidu Maps）— 中国大陆

#### 环境配置

```env
BAIDUMAP_API_KEY=
BAIDUMAP_GEOCODING_URL=https://api.map.baidu.com/geocoding/v3/
BAIDUMAP_REVERSE_GEOCODING_URL=https://api.map.baidu.com/reverse_geocoding/v3/
BAIDUMAP_PLACE_SEARCH_URL=https://api.map.baidu.com/place/v2/search
BAIDUMAP_RET_COORDTYPE=gcj02ll
```

#### 请求

- 正向地理编码：`GET {BAIDUMAP_GEOCODING_URL}?address={地址}&output=json&ak={KEY}&ret_coordtype={RET_COORDTYPE}`。`status` 为 `0` 才算成功。
- 逆地理编码：`GET {BAIDUMAP_REVERSE_GEOCODING_URL}?location={纬度},{经度}&output=json&ak={KEY}&coordtype=gcj02ll&ret_coordtype={RET_COORDTYPE}`。城市名和 `adcode` 取自 `result.addressComponent`；省辖县或县级市的 `city` 可能为空，此时用 `district`。
- 周边检索：`GET {BAIDUMAP_PLACE_SEARCH_URL}?query={关键词}&tag={分类}&location={纬度},{经度}&radius={米}&scope=2&page_size=20&filter=industry_type:cater|sort_name:distance|sort_rule:1&output=json&ak={KEY}&ret_coordtype={RET_COORDTYPE}`。`page_size` 最大 20；餐饮用 `tag=美食`；半径、检索词与次数由项目需求决定。
- 2026-11-08 实测：地理编码、周边检索、逆地理编码、轻量级步行路线四组接口均返回 `status=0`。

#### 性能要求

- 同一个地点做一次正向加一次逆地理编码，两者串行（逆地理依赖正向的坐标）。

#### 关键陷阱

- 百度默认返回 BD-09 坐标；`ret_coordtype=gcj02ll` 返回国测局坐标。接口的 `location` 参数顺序是纬度,经度，与高德相反。
- 检索结果用 GCJ-02；发给 Open-Meteo 前本地换算 WGS-84。百度不提供 WGS-84 输出（坐标转换接口转 WGS-84 返回 `status=4`）。
- Key 只放查询参数，只在服务端使用。浏览器打开 `https://api.map.baidu.com/marker?location={纬度},{经度}&title={名称}&content={内容}&output=html&coord_type=gcj02&src=webapp.{公司}.{应用}`；`coord_type=gcj02` 必传，否则位置偏移；`src` 必选。
- 需要本地时间时，用逆地理编码的 `adcode` 映射 IANA 时区：`65`→`Asia/Urumqi`，`81`→`Asia/Hong_Kong`，`82`→`Asia/Macau`，`71`→`Asia/Taipei`，其他成功结果→`Asia/Shanghai`。`adcode` 是区县级码，前两位省份足够判断；覆盖港澳台需在控制台开通全球逆地理编码服务。

### Google Maps — 中国大陆以外（含香港、澳门）

#### 环境配置

```env
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_DIRECTIONS_URL=https://maps.googleapis.com/maps/api/directions/json
GOOGLE_MAPS_PLACES_SEARCH_NEARBY_URL=https://places.googleapis.com/v1/places:searchNearby
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
GOOGLE_MAPS_PLACES_PHOTO_PREFIX=https://places.googleapis.com/v1
GOOGLE_MAPS_FIELD_MASK=places.displayName,places.formattedAddress,places.location,places.photos
```

#### 请求

- 地理编码：`GET ${GOOGLE_MAPS_GEOCODE_URL}?address={地址}&key=${GOOGLE_MAPS_API_KEY}`。`status` 为 `OK` 才算成功。坐标是 WGS-84，顺序纬度,经度（与高德相反）。2026-10-13 实测返回 `OK`。
- 路线：`GET ${GOOGLE_MAPS_DIRECTIONS_URL}?origin={纬度},{经度}&destination={纬度},{经度}&mode=walking&key=${GOOGLE_MAPS_API_KEY}`。2026-10-13 实测返回 `OK`。
- 周边检索：`POST` searchNearby，正文用 `locationRestriction.circle`；文字检索：`POST` searchText，正文用 `textQuery` 加 `locationBias.circle`。请求头 `Content-Type: application/json`、`X-Goog-Api-Key`、`X-Goog-FieldMask: ${GOOGLE_MAPS_FIELD_MASK}`。少了 `circle` 会 400。2026-10-13 实测两个正文都返回 HTTP 200 且响应里有 `places`：

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

- 地点照片：`GET ${GOOGLE_MAPS_PLACES_PHOTO_PREFIX}/{photo.name}/media?maxHeightPx=800`，请求头 `X-Goog-Api-Key`。`photo.name` 的值是 `places/{placeId}/photos/{photoId}`；必须带 `maxHeightPx` 或 `maxWidthPx`（1–4800）。默认重定向到图片，服务端跟随重定向取回字节再交给浏览器；客户端 `img` 指向自己的路由，不带 Key。2026-10-13 实测跟随重定向后为 `image/jpeg`。

#### 性能要求

- 同一地点只做一次地理编码，后面的地点检索复用这次坐标，不要再打一次。

#### 关键陷阱

- Key 放置规则：Geocoding、Directions 把 Key 放查询参数 `key`；Places 与地点照片把 Key 放请求头 `X-Goog-Api-Key`。禁止用 HTTP 引用来源限制这把 Key（服务端请求没有 Referer，会返回 `REQUEST_DENIED`）；Key 的应用限制用 IP 地址，本机开发可暂不限制；在 Google Cloud 只启用实际调用的 API。
- Geocoding、Directions 与 Places 的 host 不同，禁止把 `https://maps.googleapis.com` 当成统一 base。
- 是否在中国大陆看地理编码结果里 `country` 的短码：`CN` 是中国大陆，`HK`、`MO`、`TW` 和其他国家不是；浏览器坐标先做一次逆地理编码，不写经纬度方框。
- 浏览器打开 `https://www.google.com/maps/dir/?api=1&destination=纬度,经度`，不加载 JS API。

### 天气

#### 环境配置

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

#### 请求

`GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m&timezone=auto`。无需 Key，坐标为 WGS-84。2026-10-13 实测返回含上述四项的 `current`。`current` 的参数与 `timezone`（`auto` 或项目得到的 IANA）按该项目需求取用，最少需要 `temperature_2m` 与 `weather_code`。

#### 性能要求

- 用已经得到的坐标发一次预报，不要为这次预报重新定位。

#### 关键陷阱

- `weather_code` 是 WMO 代码，展示前映射为目标语言文案；禁止把裸代码或英文描述直接给用户。

### 星座 MCP

#### 环境配置

```env
# Streamable HTTP MCP，无需 Key
PANDORIUM_MCP_URL=https://pandorium.app/mcp
```

#### 请求

项目需求写明星座服务时才接入。`POST $PANDORIUM_MCP_URL`，请求头 `Content-Type: application/json` 与 `Accept: application/json, text/event-stream`。2026-10-13 实测：下面的 `tools/call` 返回 SSE，`event: message` 后的 `data:` 行为 JSON-RPC，`result.content` 为文本；`tools/list` 的 `inputSchema` 要求 `birthData`，且 `location` 必填 `latitude`、`longitude`、`timezone`、`city`。

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

- 成功时解析 SSE 的 `data:` 行：`result.content` 是数组，正文在 `result.content[0].text`。没有 `content` 即失败。
- `get_current_transits` 使用同一个 `birthData`，`params.name` 改为该工具名。
- 无出生时刻时 `time` 固定 `12:00`。

#### 性能要求

- 使用已经解析的出生城市坐标发一次 `calculate_natal_chart`。
- 与天气是否并行、同一档案同一日请求几次，由该项目需求决定。

#### 关键陷阱

- 禁止只传出生日期：`location` 缺 `city` 或 IANA `timezone` 会被 schema 拒绝。
- 429 响应体为 `{"success":false,"error":"Too many requests","retryAfter":…}`，不是 JSON-RPC；须展示 `retryAfter`，禁止对 429 自动重试。
- 测试用夹具，不出网打 live MCP；冒烟间隔至少 30 秒。

### 宝可梦 TCG 目录

#### 环境配置

```env
# GET {TCGDEX_BASE_URL}/{lang}/cards/{id}，无需 Key
# lang：zh-cn、zh-tw、en；例 /zh-cn/cards/SV7-001
TCGDEX_BASE_URL=https://api.tcgdex.net/v2
```

#### 请求

- 项目需求写明卡片目录时才接入。`GET {TCGDEX_BASE_URL}/{lang}/cards/{id}`。
- 2026-10-13 实测 `GET /v2/zh-cn/cards/SV7-001` 与 `GET /v2/en/cards/base1-4` 返回 200；路径里的 `zh` 会返回 `language-invalid`。
- 2026-11-13 实测（中国大陆网络）：`api.tcgdex.net` 间歇性 15 秒级无响应；图片源站 `assets.tcgdex.net` 为单台 OVH 服务器、无 CDN，404 的 TTFB 也达 8 秒，大图请求 20 秒以上超时。

#### 性能要求

- 按卡片 id 发一次 GET，不要对同一张卡重复请求。
- 卡牌图片与元数据经应用后端代理并永久缓存（卡牌图片内容不变），前端不直连 `assets.tcgdex.net`；列表用缩略图（`low`），详情才用大图（`high`）。

#### 关键陷阱

- `lang` 只用 `zh-cn`、`zh-tw` 或 `en`；只使用文档记载的路径与字段。

### 中式菜谱 HowToCook — 本地 stdio MCP

#### 环境配置

无需密钥。服务端以本地子进程运行：

```bash
npx -y howtocook-mcp
```

stdio 通信：先发 `initialize`，再发 `notifications/initialized`，然后 `tools/list`、`tools/call`；每行一个 JSON-RPC 消息。

##### 请求

2026-11-13 实测（版本 0.2.2，协议 2024-11-05，启动时载入 322 个菜谱）。工具名带前缀 `mcp_howtocook_`：

- `getAllRecipes`：无实际参数（schema 要求一个占位 `no_param`）
- `getRecipesByCategory`：`category` 必填，枚举：水产、早餐、调料、甜品、饮品、荤菜、半成品加工、汤、主食、素菜
- `recommendMeals`：`peopleCount` 必填（1–10），可选 `allergies`、`avoidItems`
- `whatToEat`：`peopleCount` 必填
- `getRecipeById`：按 id 取单道菜

返回在 `result.content[0].text`，是 JSON 数组，元素含 `id`（形如 `dishes-aquatic-咖喱炒蟹`）、`name`、`description`（Markdown 正文）等字段。

#### 性能要求

- 服务子进程常驻复用，不要每次调用都重启。
- 瓶颈不在响应耗时（数据在本地，2026-11-13 实测每次调用都在 0.01 秒内），在返回体积占用的上下文。实测各工具返回体积：`getRecipeById` 约 3.4KB（单道菜，最小）；`whatToEat` 约 6.5KB；`getAllRecipes` 约 43.5KB（322 个菜谱全量）；`getRecipesByCategory` 单个分类即可达约 40KB（水产）；`recommendMeals` 约 137KB（一周膳食计划加购物清单，体积最大）。
- 优先 `getRecipeById`（支持按名称模糊匹配）与 `getRecipesByCategory`；确需列表时，先想清楚全量或整周计划是否真要进上下文；`getAllRecipes` 官方文档自注「慎用这个——上下文太大」。

#### 关键陷阱

- `getRecipeById` 的参数名是 `query`（菜谱名称或 ID，支持模糊匹配）；传 `id`、`name` 等其他参数名会被 schema 拒绝（`-32602`）。
- 版本差异：0.2.2 已把菜谱数据随包内置（启动即载入 322 个菜谱）；早期版本启动时从远程 `weilei.site` 拉取 `all_recipes.json`，偶发 10 秒连接超时导致服务退出（官方 issue #11）。生产环境固定版本，避免行为漂移。
- 原文是中文。目标语言不是中文时先翻译再展示；翻译缓存与标注规则由该项目需求决定。

### 西式菜谱 RecipeWiki — 远程 MCP

#### 环境配置

```env
# Streamable HTTP MCP，无需 Key、无需账号
RECIPEWIKI_MCP_URL=https://www.recipewiki.com/api/mcp
```

#### 请求

`POST $RECIPEWIKI_MCP_URL`，请求头 `Content-Type: application/json` 与 `Accept: application/json, text/event-stream`。2026-11-13 实测（协议 2025-03-26，服务端 recipewiki 1.0.0）：响应是普通 `application/json` 的 JSON-RPC，不是 SSE；无需会话头。工具：

- `search_recipes`：参数 `query`、`tags`、`cuisine`、`course`、`sort`、`page`、`limit`，全部可选。返回摘要（标题、链接、图、时间、份数、过敏原），不含食材与步骤，取详情要再调 `get_recipe`。
- `get_recipe`：`slug` 必填。返回 schema.org Recipe 的 JSON 字符串（在 `result.content[0].text`），字段含 `recipeIngredient`、`recipeInstructions`、`prepTime`、`cookTime`、`totalTime`、`recipeYield`。
- `list_tags`：可选 `group`。
- `list_allergens`：无参数。

#### 性能要求

- 列表与详情是两次调用：先 `search_recipes`，再对选中的 `slug` 调 `get_recipe`。

#### 关键陷阱

- `query` 会忽略 `tags`、`cuisine`、`course`（服务端在工具说明里明确），不要组合使用还指望两者都生效。
- 过敏原字段按子菜谱向上汇总；下「安全」结论前先看 `list_allergens` 的说明。

### 西式菜谱 TheMealDB — REST API

#### 环境配置

```env
# 路径中的 1 是官方公开测试 Key，无需申请
THEMEALDB_BASE_URL=https://www.themealdb.com/api/json/v1/1
```

#### 请求

2026-11-13 实测全部返回 HTTP 200：

- 按名称搜索：`GET {THEMEALDB_BASE_URL}/search.php?s={名称}`
- 按 id 取详情：`GET {THEMEALDB_BASE_URL}/lookup.php?i={idMeal}`
- 随机一道：`GET {THEMEALDB_BASE_URL}/random.php`
- 分类列表：`GET {THEMEALDB_BASE_URL}/list.php?c=list`
- 按分类过滤：`GET {THEMEALDB_BASE_URL}/filter.php?c={分类}`

响应是 `{"meals":[…]}`，字段含 `idMeal`、`strMeal`、`strCategory`、`strArea`、`strInstructions`、`strMealThumb`。

#### 性能要求

- 每次请求取一次；同一菜谱不重复请求。

#### 关键陷阱

- 原文是英文。目标语言不是英文时先翻译再展示；翻译缓存与标注规则由该项目需求决定。
- `filter.php` 只返回缩略列表（无步骤与食材），详情用 `lookup.php`。

## 4.安装

### npm 镜像

准备工程时按当前位置安装依赖。尚无 `package.json` 时，先按项目需求建工程再安装；已有 lockfile 时用 `npm ci`，不要用 `npm install` 改写 lockfile。

中国大陆、香港和澳门：

```bash
# 还没有 lockfile 时，只用来生成它
npm install --registry=https://registry.npmmirror.com
# 已有 lockfile 时
npm ci --registry=https://registry.npmmirror.com
```

其他地区：

```bash
# 还没有 lockfile 时，只用来生成它
npm install
# 已有 lockfile 时
npm ci
```

- 已有 lockfile 时只用 `npm ci`，不要用 `npm install` 改写它，否则 `resolved` 会变成镜像地址。lockfile 里的 `registry.npmjs.org` 表示「当前配置的 registry」，`npm ci --registry=...` 从镜像下载、不改 lockfile。
- 禁止 `npm config set registry`，禁止提交项目 `.npmrc`。
- `npm ci` 遇 `edgesOut` 空指针时，加 `--legacy-peer-deps` 再装一次；其他 peer 依赖失败同理。不要改 lockfile 里的版本，不要为此改 `@types/node`。
- 若 `npm config get registry` 不是 `https://registry.npmjs.org/`，用 `npm config delete registry` 清掉用户级设置后再 `npm ci`。

### 密钥

- 密钥在仓库根的 `.secrets`。这个文件由人事先放好，真值只在这里。
- 开工时只复制本项目用到的变量名，从仓库根 `.secrets` 写入该工程 `.env.local`；`.secrets` 里多出来的变量不写。需要的变量不存在或为空则停止。
- 运行时只读 `.env.local`。禁止 `NEXT_PUBLIC_*` 暴露密钥。模型 id 用环境变量锁死，不依赖提供方默认模型。
- 回复、日志和浏览器响应里不出现密钥。`.secrets` 和 `.env.local` 都不提交。

## 5.调用顺序

彼此独立的调用同时发出。一块完成就显示这一块。需求写明的先后依赖照需求。失败只说明这一块。不自动重试。

## 6.测试

本节只写工具和跑法。业务用例、夹具和通过标准写在该项目的测试文件里，不写在需求里，也不写在这里。

- 单元和组件用 Vitest。要渲染页面时用 `jsdom` 和 Testing Library。
- 接口和集成用同一套 Vitest，请求打到本机路由。外部地图、对话和图像用夹具，不出网。
- 测试文件点名的浏览器旅程用 Playwright。没有点名就不跑端到端。
- 至少一条关键用户旅程用真实浏览器 E2E 覆盖。
- 测试数据放临时目录（可用 `DATA_DIR` 指定），不用项目的 `data/`。密钥不进测试代码，不进日志。
