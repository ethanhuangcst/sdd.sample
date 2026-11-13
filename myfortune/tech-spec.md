# 技术规格

- 本文件只保留本项目用到的能力；能力全集见仓库根的 `tech-spec-cn.md`。产品行为写在 `req-spec-cn.md`。
- 第三方调用失败要向界面说明原因。禁止静默空成功、空图、空列表。需求没写重试时，用户可以手动再试一次。禁止自动循环重试。
- 降级只在上游失败、或用户可见等待结束后仍为空时出现；禁止在等待期间静默切降级。
- 项目需求里的安装清单或测试章短于本文件时，以需求为准。
- 用户可见文案是否使用 i18n，由项目需求决定。未要求时可以直接写一种语言。

## 1.原则

- 项目文档在仓库根的 `<项目>/` 目录。代码按类型分目录：Next.js 用 `web/`。
- 本地文件存储：持久化用本地文件，放在项目的 `data/`。不使用数据库，不使用 ORM。

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

### 塔罗 MCP

本地 stdio MCP，无需 Key、无网络依赖。两个候选实测后推荐 `tarotoo-mcp-server`（另一个不可用，见关键陷阱）。

#### 环境配置

```bash
npx -y tarotoo-mcp-server
```

安装走 npmmirror 镜像：默认 npmjs 源在大陆网络不可达，`npx` 会在下载阶段卡住数分钟且无任何输出。

#### 请求

2026-11-13 实测（tarotoo-tarot 2.0.1，协议 2024-11-05，78 张 Rider–Waite–Smith 牌义数据集随包内置，MIT）。返回都在 `result.content[0].text`，JSON 格式：

- `get_card_meaning`：`name` 必填（如 `"fool"`，匹配宽容）。返回单张完整牌义：正逆位含义与关键词、爱情/事业/情绪/灵性语境、元素、星象、正逆位是否牌。实测 `fool` 约 1.4KB。
- `draw_cards`：`count`（1–10，默认 1）、`allow_reversed`。随机抽不重复的牌并带正逆位。实测抽 3 张约 1.9KB。
- `list_cards`：`arcana?`（major/minor）、`suit?`（wands/cups/swords/pentacles）。
- `search_cards`：`query` 必填、`limit?`（1–78，默认 10），按主题找牌（如 `"heartbreak"` 命中宝剑三）。
- `yes_no_answer`：`name` 必填，返回是否牌结论与依据。

#### 性能要求

- 数据集随包内置、无网络调用，单次调用毫秒级。
- `tools/list` 冷启动可能 16–90 秒才响应，握手写法见关键陷阱；建议服务子进程常驻复用。

#### 关键陷阱

- **握手方式敏感**：`initialize`、`notifications/initialized`、`tools/list` 必须连续写入（官方 MCP 客户端 SDK 的行为）。逐条等待式握手（收到 `initialize` 响应后再发下一条）实测会让 `tools/list` 长时间无响应。自写客户端用官方 SDK 或连发模式。
- 数据集只有 Rider–Waite–Smith 体系；牌义仅供娱乐与自省（数据集官方声明），不构成医疗、法律、财务建议。
- 备选 `tarot-mcp`（v0.1.0）：连发模式下 `tools/list` 也无响应，判不可用。

### 八字 MCP

本地确定性计算，无需 Key、不出网。推荐 `@openfate/bazi-mcp`；备选 `shunshi-bazi-mcp`（可用但冷启动约 48 秒）。

#### 环境配置

```bash
npx -y @openfate/bazi-mcp
```

安装同样走 npmmirror。

#### 请求

2026-11-13 实测（openfate-bazi-mcp 0.3.1，协议 2024-11-05，本地计算不出网，MIT）：

- `calculate_bazi_chart`：确定性四柱排盘。参数：`year`、`month`、`day`、`hour`、`minute`、`second`、`gender`、`calendarType`、`isLeapMonth`、`longitude`、`timezone`、`timezoneId`、`dstOffset`、`enableTrueSolarTime`、`dayBoundaryMode`。实测 1990-03-15 12:00 上海男命、`dayBoundaryMode:"ZI_HOUR_23"` 返回约 10.3KB 结构化命盘（`data` + `attribution`）。
- `detect_bazi_interactions`：地支冲合刑会破害（本命盘、流年、合盘）。
- `calculate_true_solar_time`：真太阳时的计算与解释。
- `reverse_bazi_to_solar_times`：由四柱反查公历候选时间。
- `get_openfate_bazi_policy`：换日、夏令时、真太阳时的计算口径。

#### 性能要求

- 本地确定性计算，毫秒级返回；不把命盘请求发往任何远端。
- `tools/list` 冷启动同样可能数十秒无响应，握手要求与塔罗相同；服务子进程常驻复用。

#### 关键陷阱

- `dayBoundaryMode` 必须用枚举值 `'MIDNIGHT_00'` 或 `'ZI_HOUR_23'`；传普通字符串会被 schema 拒绝（`-32602`，探针实测）。
- 子时换日口径（`dayBoundaryMode`）与真太阳时开关（`enableTrueSolarTime`）直接影响时柱；项目需求必须写明采用哪种口径。
- LLM 不得自行推算历法（节气边界、真太阳时、农历转换、地支互动）；一律调用引擎，这是该包的官方立场。
- 备选 `shunshi-bazi-mcp`（v0.1.0）：工具 `getBaziChart`、`getHuangli`（黄历宜忌）；参数为数值字段；默认真太阳时开启；实测冷启动约 48 秒才响应 `tools/list`。



## 4.安装

### npm 镜像

#### 原则

- 尚无 `package.json` 时，先按项目需求建工程再安装。
- 已有 lockfile 时只用 `npm ci`，不要用 `npm install` 改写它，否则 `resolved` 会变成镜像地址。lockfile 里的 `registry.npmjs.org` 表示「当前配置的 registry」，`npm ci --registry=...` 从镜像下载、不改 lockfile。
- 禁止 `npm config set registry`，禁止提交项目 `.npmrc`。
- `npm ci` 遇 `edgesOut` 空指针时，加 `--legacy-peer-deps` 再装一次；其他 peer 依赖失败同理。不要改 lockfile 里的版本，不要为此改 `@types/node`。
- 若 `npm config get registry` 不是 `https://registry.npmjs.org/`，用 `npm config delete registry` 清掉用户级设置后再 `npm ci`。

#### 中国大陆、香港和澳门

```bash
# 还没有 lockfile 时，只用来生成它
npm install --registry=https://registry.npmmirror.com
# 已有 lockfile 时
npm ci --registry=https://registry.npmmirror.com
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
