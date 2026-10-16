# MyDay 中文演示技术规格（Google Maps）

只服务中国大陆以外这一次中文演示。一次运行只读这一份。不包含高德、星座、宝可梦、路线和地点照片。

## 技术栈

| 用途 | 技术 |
| --- | --- |
| 应用 | Next.js 16、React 19、TypeScript、Tailwind CSS 4 |
| 对话 | `openai` SDK，Quanzil chat completions |
| 图像 | Gemini `generateContent`，不用 `openai` 的 images 方法 |
| 测试 | Vitest |

不安装 React Query、Zustand、RHF、Playwright。不写 `data/`。工程在工作区的 `web/`。

## 安装

尚无 `web/package.json` 时，先在空的 `web/` 建工程，再安装。若当时 `web/` 里只有 `.env.local`，先移开，建完再移回，内容不改。

- 中国大陆、香港、澳门：`npm install --registry=https://registry.npmmirror.com`
- 其他地区：`npm install`
- 已有 lockfile 时用对应的 `npm ci`，不要用 `npm install` 改写 lockfile
- peer 依赖失败时，加上 `--legacy-peer-deps` 再装一次。不要改 `@types/node` 的版本

## 密钥

密钥在 `web/.env.local`。这个文件由人事先放好。不要读 `specs/.secrets`，不要新建、改写或复制 `.env.local`。文件不存在，或本需求用到的变量为空，则停止。禁止 `NEXT_PUBLIC_*`。回复、日志和浏览器响应里不出现密钥。

## 对话

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6-astra
```

用 `openai` SDK，创建时传入上面的 Key 和 base URL。不要再设置 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL`。用 `max_completion_tokens`，不用 `max_tokens`。`choices[0].message.content` 为空即失败。

## 图像

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

`POST` 该 URL。请求头 `Authorization: Bearer <QUANZIL_API_KEY>`。正文带 `generationConfig.responseModalities: ["IMAGE"]`。有照片时 `parts` 含文本和 `inlineData`。`mimeType` 用照片本身的类型：`image/png`、`image/jpeg`、`image/webp`、`image/heic`、`image/heif`。先把长边缩到约 1024px。1×1 图片会 400。不要设置 `imageConfig.imageSize`。结果在 `candidates[0].content.parts[].inlineData`。没有 `inlineData` 即失败。读不到照片就报错，禁止改成不带照片的请求。

## Google Maps

```env
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
```

- 地理编码：`GET ${GOOGLE_MAPS_GEOCODE_URL}?address={地址}&key=`。`status` 为 `OK` 才算成功。坐标是 WGS-84，顺序是纬度,经度
- 餐厅：`POST` searchText。请求头 `Content-Type: application/json`、`X-Goog-Api-Key`、`X-Goog-FieldMask: places.displayName,places.formattedAddress,places.location`
- 正文是 `textQuery` 加 `locationBias.circle`。`circle` 不能省。半径 5000 米。只发这一次
- 名称和地址用这一次的返回。距离用返回坐标和已有坐标计算，不另发请求。不按行再请求详情
- 浏览器打开 `https://www.google.com/maps/dir/?api=1&destination=纬度,经度`。不加载 JS API
- Key 不放进浏览器

## 天气

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

`GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,weather_code`。坐标为 WGS-84。`weather_code` 先变成中文再显示。用已经得到的坐标，不要为了天气再定位一次。

## 调用顺序

彼此独立的调用同时发出。一块完成就显示这一块。穿搭图等待穿搭文字，这是需求里的依赖，不是把其他对话排在它前面。失败只说明这一块。不自动重试。
