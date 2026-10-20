# MyDay 技术规格


##  1.原则
- 在工作区根目录工作。不要参考其他项目。不要把代码或文档写到本工作区以外。
- 工作区根目录只放两样：`specs/`，以及工程目录 `web/`。Next.js 代码只放在 `web/`。
- 不做数据存储，不写 `data/`。
- 只安装必需依赖项，不安装 React Query、Zustand、RHF、Playwright。

## 2.技术栈

| 用途 | 技术 |
| --- | --- |
| 应用 | Next.js 16、React 19、TypeScript、Tailwind CSS 4 |
| 对话 | `openai` SDK，Quanzil chat completions |
| 图像 | Gemini `generateContent`，不用 `openai` 的 images 方法 |
| 测试 | Vitest |


## 3.能力

### 对话

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6-astra
```

用 `openai` SDK，创建时传入上面的 Key 和 base URL。不要再设置 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL`。用 `max_completion_tokens`，不用 `max_tokens`。`choices[0].message.content` 为空即失败。

`gpt-6-astra` 会先把推理 token 计入 `max_completion_tokens`，再写 `choices[0].message.content`。需求提示词里的字数和字符数不设置这个上限。状态总结、行程、穿搭这三次对话都把 `max_completion_tokens` 设为 1200。`choices[0].message.content` 为空即失败，包括 `finish_reason` 为 `length`、推理用完上限、正文为空的情况。

### 图像

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

`POST` 该 URL。请求头 `Authorization: Bearer <QUANZIL_API_KEY>`。正文带 `generationConfig.responseModalities: ["IMAGE"]`。有照片时 `parts` 含文本和 `inlineData`。`mimeType` 用照片本身的类型：`image/png`、`image/jpeg`、`image/webp`、`image/heic`、`image/heif`。照片是小图：上传由用户控制，长边约 2048px 以内、文件 2MB 以内，应用不做缩图。生图请求加 75 秒超时，超时按该区域失败处理，不重试。1×1 图片会 400。不要设置 `imageConfig.imageSize`。结果在 `candidates[0].content.parts[].inlineData`。没有 `inlineData` 即失败。读不到照片就报错，禁止改成不带照片的请求。

### Google Maps

```env
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
```

- 地理编码：`GET ${GOOGLE_MAPS_GEOCODE_URL}?address={地址}&key=`。`status` 为 `OK` 才算成功。坐标是 WGS-84，顺序是纬度,经度。
- 餐厅：`POST` searchText。请求头 `Content-Type: application/json`、`X-Goog-Api-Key`、`X-Goog-FieldMask: places.displayName,places.formattedAddress,places.location`
- 正文是 `textQuery` 加 `locationBias.circle`。`circle` 不能省。半径 5000 米。只发这一次。
- 名称和地址用这一次的返回。距离用返回坐标和已有坐标计算，不另发请求。不按行再请求详情。
- 浏览器打开 `https://www.google.com/maps/dir/?api=1&destination=纬度,经度`。不加载 JS API。
- Key 不放进浏览器，只在服务端使用。禁止 `NEXT_PUBLIC_*`。

### 时区

地理编码成功后，从 `address_components` 里 `country` 的 `short_name` 映射 IANA 时区 id。不请求时区接口，不安装 `geo-tz`、`tz-lookup` 或同类包。

| 国家短码 | IANA |
| --- | --- |
| TW | `Asia/Taipei` |
| HK | `Asia/Hong_Kong` |
| MO | `Asia/Macau` |
| SG | `Asia/Singapore` |
| JP | `Asia/Tokyo` |
| KR | `Asia/Seoul` |
| AU | `Australia/Sydney` |
| GB | `Europe/London` |
| FR | `Europe/Paris` |
| DE | `Europe/Berlin` |
| US | `America/Los_Angeles` |
| 其他已成功 | `Asia/Taipei` |

- 用 `Intl.DateTimeFormat`，`timeZone` 设为上表的 IANA，算出该地点的当地日期和时间。不要用浏览器系统时区代替。不要用经度除以 15 估算。
- 周岁里的「今天」用这个 IANA 的当地日历日期。未到今年生日则减一岁。
- 进入第二页时带上这个 IANA、当地时间和坐标。

### 天气

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

- `GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,weather_code&timezone={IANA}`。坐标为 WGS-84。`timezone` 用上一节得到的同一个 IANA。用已经得到的坐标，不要为了天气再定位一次。
- `weather_code` 按下表译成中文后再显示。表里没有的显示「未知」。不要为了翻译再请求别的接口。

| 代码 | 中文 |
| --- | --- |
| 0 | 晴 |
| 1 | 大部晴朗 |
| 2 | 局部多云 |
| 3 | 阴 |
| 45 | 雾 |
| 48 | 雾凇 |
| 51 | 小毛毛雨 |
| 53 | 毛毛雨 |
| 55 | 大毛毛雨 |
| 56 | 冻毛毛雨 |
| 57 | 强冻毛毛雨 |
| 61 | 小雨 |
| 63 | 中雨 |
| 65 | 大雨 |
| 66 | 冻雨 |
| 67 | 强冻雨 |
| 71 | 小雪 |
| 73 | 中雪 |
| 75 | 大雪 |
| 77 | 雪粒 |
| 80 | 小阵雨 |
| 81 | 阵雨 |
| 82 | 强阵雨 |
| 85 | 小阵雪 |
| 86 | 阵雪 |
| 95 | 雷暴 |
| 96 | 雷暴伴小冰雹 |
| 99 | 雷暴伴大冰雹 |

### 星座

不调用星座服务，不安装星座包。只看公历月日，不考虑出生时刻和年份。界日与 `specs/req-spec.md` 一致：摩羯 12/22–1/19，水瓶 1/20–2/18，双鱼 2/19–3/20，白羊 3/21–4/19，金牛 4/20–5/20，双子 5/21–6/21，巨蟹 6/22–7/22，狮子 7/23–8/22，处女 8/23–9/22，天秤 9/23–10/23，天蝎 10/24–11/22，射手 11/23–12/21。

- 在 `web/` 里写本地函数。12/21、12/22、1/19、1/20 以及其余每个界日的前后一天都要有单元测试。
- 界面显示中文星座名。

## 4.安装

### 第一步：安装依赖项

- 尚无 `web/package.json` 时，在工作区根目录执行下面这一条。`web/` 不存在或是空目录都可以。不要停在创建工具的提问上。中国大陆、香港、澳门：
  ```
  npm_config_registry=https://registry.npmmirror.com npx create-next-app@16 web --yes --ts --tailwind --eslint --app --no-src-dir --use-npm
  ```
  其他地区：同一条命令，不设置 `npm_config_registry`。
  这条命令会自己安装 Next.js 依赖。不要在它之后再无目的地执行 `npm install`。
- 然后在 `web/` 安装本项目额外的包。中国大陆、香港、澳门加上 `--registry=https://registry.npmmirror.com`：
  ```
  npm install openai
  npm install -D vitest jsdom @testing-library/react @testing-library/jest-dom
  ```

- 已有 `web/package.json` 和 `web/package-lock.json` 时，不要再跑 `create-next-app`。在 `web/` 执行 `npm ci`。不要用 `npm install` 改写已有 lockfile。lockfile 里还没有上面这些包时，再按那两条安装命令补上。
- peer 依赖失败时，给失败的那条 `npm install` 加上 `--legacy-peer-deps` 再装一次。不要改 `@types/node` 的版本。

### 第二步：生成密钥

- 密钥在 `specs/.secrets`。这个文件由人事先放好。
- 只复制「3.能力」代码块里的变量名，从 `specs/.secrets` 写入 `web/.env.local`。`.secrets` 里多出来的变量不写。需要的变量不存在或为空则停止。禁止 `NEXT_PUBLIC_*`。回复、日志和浏览器响应里不出现密钥。

## 5.调用顺序

彼此独立的调用同时发出。一块完成就显示这一块。穿搭图等待穿搭文字，这是需求里的依赖，不是把其他对话排在它前面。失败只说明这一块。不自动重试。
