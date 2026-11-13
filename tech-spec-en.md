# Tech spec

- This file lists the full set of supported capabilities. Product behavior lives in each project's requirement files.
- When using this file, delete the capability sections the project does not need and keep only what it uses, so unrelated capabilities do not pollute the context.
- Later projects may include a web app, MCP, RAG, and an agent. This file sets the stack, third-party capabilities, and install steps those parts use. Do not wire a capability the project requirements do not name.
- A third-party failure must tell the UI why. No silent empty success, empty image, or empty list. When the requirements do not mention retry, the user may retry once by hand. Do not retry in a loop.
- A fallback may appear only after an upstream failure, or after a user-visible wait ends with nothing; never switch to a fallback silently during the wait.
- When the install list or the test section in the project requirements is shorter than this file, follow the requirements.
- Whether user-facing copy uses i18n is decided by that project's requirements. When they do not require it, one language written in the UI is allowed.

## 1. Principles

- Project documents live in `pack/projects/<project>/`. Code is organized by type: Next.js in `web/`, RAG in `rag/`, MCP in `mcp/`.
- Minimal dependencies: section 2 lists the technologies you may use; not every project installs all of them. Install only the packages the project requirements name. `create-next-app` picks and installs TypeScript.
- Local file storage: persistence uses local files in the project's `data/`. No database, no ORM.
- When one region has several providers (AMap and Baidu for maps), use the one the project requirements name.

## 2. Stack

Majors below (reference date 2026-10). The exact patch follows the project's `package.json` and lockfile. TypeScript 7.0 is the target; when `create-next-app` creates the project, use the TypeScript that command installs. Do not change it to 7.0 while creating the project.

| Category | Tech | Major |
| --- | --- | --- |
| Framework | Next.js (App Router) | 16.4 |
| UI / language | React, TypeScript | 19.3, 7.0 |
| Styles | Tailwind CSS | 4.3 |
| State / forms | React Query, Zustand, RHF + Zod + `@hookform/resolvers` | 5.x, 5.x, 7.x + 4.x |
| Storage | Local JSON (`data/`), no database | — |
| AI SDK | `openai` SDK for chat only; image uses Gemini `generateContent` (see section 3) | 7.x (6→7 has breaking changes; follow the official migration guide when upgrading) |
| Test | Vitest, RTL, Playwright | 5.x, 16.x, 1.64 |

## 3. Capabilities

Each capability is recorded in four parts: environment, request (with test records), performance, and pitfalls. Test dates sit next to the findings.

When using this file, delete the capability subsections this project does not need, so unrelated capabilities stay out of the context.

### Chat LLM

#### Environment

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6.1-sol
```

#### Request

The provider is Quanzil, OpenAI-compatible chat completions, via the `openai` SDK. Tested 2026-10-13: the call below returned a non-empty `choices[0].message.content`.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.QUANZIL_API_KEY,
  baseURL: process.env.QUANZIL_BASE_URL,
});

const completion = await client.chat.completions.create({
  model: process.env.QUANZIL_CHAT_MODEL,
  messages: [{ role: "user", content: "task brief" }],
  max_completion_tokens: 1024,
});

const text = completion.choices[0]?.message?.content ?? "";
```

An empty `text` is failure. `gpt-6.1-sol` counts reasoning tokens inside `max_completion_tokens` before writing the body; the word and character limits in the requirement prompts do not set this cap. When the requirement asks for one text output, set `max_completion_tokens` to 1200. An empty body is failure, including when `finish_reason` is `length`, when reasoning used the whole cap, or when the body is empty.

#### Performance

- One short chat call. Within the same user wait, do not order chat before image generation.

#### Pitfalls

- Pass `QUANZIL_API_KEY` and `QUANZIL_BASE_URL` into the client constructor. Do not also set `OPENAI_API_KEY` or `OPENAI_BASE_URL`: the SDK reads them by default and calls `api.openai.com`.
- Use `max_completion_tokens`, not `max_tokens`.

### Image generation LLM

#### Environment

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

Shares `QUANZIL_API_KEY` with chat. The request is native Gemini `generateContent`, not the `openai` SDK images methods.

#### Request

`POST $QUANZIL_IMAGE_URL` with headers `Authorization: Bearer <QUANZIL_API_KEY>` and `Content-Type: application/json`:

```json
{
  "contents": [{
    "role": "user",
    "parts": [
      { "text": "task brief" },
      { "inlineData": { "mimeType": "image/png", "data": "<base64>" } }
    ]
  }],
  "generationConfig": { "responseModalities": ["IMAGE"] }
}
```

- The reference image's `mimeType` is the photo's own type: `image/png`, `image/jpeg`, `image/webp`, `image/heic`, or `image/heif`. Do not force JPEG.
- The photo upload is user-controlled: long edge within about 2048px and file within 2MB. The app does not scale images.
- Without a user photo, `parts` keeps only `text`; with a photo, `inlineData` is required. A photo that cannot be read is an error; never resend without the photo.
- Tested 2026-10-13: a text-only request and a 64×64 JPEG `inlineData` both returned HTTP 200 with `inlineData.mimeType` `image/jpeg`; a 1×1 JPEG returned 400 `Unable to process input image` — the reference image must be a decodable photo.
- The result is in `candidates[0].content.parts[].inlineData` (base64), with the type in the sibling `mimeType`. No `inlineData` is failure.
- Wrap the image POST in a 75-second timeout; a timeout fails that block, no retry.

#### Performance

- Send as soon as the photo and the short prompt are ready. Within the same wait, do not chain one more chat call before the image.
- The image waits on its own. Do not cut a running image generation with a short map or weather wait.
- The output is about 1024×1024; shrinking the reference image further does not shorten generation.

#### Pitfalls

- `generationConfig.responseModalities: ["IMAGE"]` is required; without it the model often returns text only.
- The model id must appear in this key's `GET /v1/models`.
- Do not set `imageConfig.imageSize`; `512` returns 400.
- When text is already available, do not put the text LLM and the image on the same critical path.

### Maps

One region may have several providers (mainland China has AMap and Baidu); use the one the project requirements name. Rules shared by all three: the key stays on the server, no `NEXT_PUBLIC_*`; the browser opens each provider's own link, no JS API; geocode the same place once; radius and search count come from that project's requirements — when the requirements do not say the range may grow, do not increase the radius and search again.

#### AMap — mainland China

##### Environment

```env
AMAP_API_KEY=
AMAP_HOST=https://restapi.amap.com
# GET {AMAP_HOST}/v3/geocode/geo?address={address}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/text?keywords={keywords}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/around?location={lng},{lat}&radius=3000&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/direction/walking?origin={lng},{lat}&destination={lng},{lat}&key=${AMAP_API_KEY}
```

##### Request

- `location`, `origin`, `destination` are longitude,latitude. Success is HTTP 200 with `status` `1` and `infocode` `10000`. Tested 2026-10-13 on all four paths with the query strings above; every `infocode` was `10000`.
- Geocoding lives under `/v3`; keyword search, nearby search, and walking routes live under `/v5`. Do not set a single `/v3` base URL that covers every endpoint.
- Nearby restaurant search (when the project finds nearby restaurants by cuisine):
  - Use `GET /v5/place/around`, not `/v5/place/text`. `location` is GCJ-02 longitude,latitude; `radius` is meters; the dining type is `types=050000`; the query term is the user's text.
  - Stop at the first result set: first the term plus `types=050000`, then the term only, then `types=050000` only.

##### Performance

- Geocode the same place once. Weather and restaurants use the current location's coordinates; the birth city is a different place and is geocoded on its own.

##### Pitfalls

- The key goes in the query parameter `key`, not a header. The browser opens `https://uri.amap.com/marker?position=longitude,latitude`.
- Browser location and Open-Meteo use WGS-84; AMap REST uses GCJ-02. Before sending coordinates to Open-Meteo, convert with the public local iteration; do not install a coordinate library. Coordinates outside China need no conversion. Nearby search keeps the original GCJ-02.
- Decide mainland China by the returned `adcode`: codes starting with 71, 81, and 82 are Taiwan, Hong Kong, and Macau — not mainland China. No result counts as a geocoding failure. For a browser location, call `GET /v3/geocode/regeo?location={longitude},{latitude}` first, then judge by `adcode`; do not use a lat/lng bounding box.

#### Baidu (Baidu Maps) — mainland China

##### Environment

```env
BAIDUMAP_API_KEY=
BAIDUMAP_GEOCODING_URL=https://api.map.baidu.com/geocoding/v3/
BAIDUMAP_REVERSE_GEOCODING_URL=https://api.map.baidu.com/reverse_geocoding/v3/
BAIDUMAP_PLACE_SEARCH_URL=https://api.map.baidu.com/place/v2/search
BAIDUMAP_RET_COORDTYPE=gcj02ll
```

##### Request

- Geocoding: `GET {BAIDUMAP_GEOCODING_URL}?address={address}&output=json&ak={KEY}&ret_coordtype={RET_COORDTYPE}`. Success is `status` `0`.
- Reverse geocoding: `GET {BAIDUMAP_REVERSE_GEOCODING_URL}?location={lat},{lng}&output=json&ak={KEY}&coordtype=gcj02ll&ret_coordtype={RET_COORDTYPE}`. City name and `adcode` come from `result.addressComponent`; for a province-administered county or county-level city, `city` may be empty — use `district` then.
- Nearby search: `GET {BAIDUMAP_PLACE_SEARCH_URL}?query={term}&tag={category}&location={lat},{lng}&radius={meters}&scope=2&page_size=20&filter=industry_type:cater|sort_name:distance|sort_rule:1&output=json&ak={KEY}&ret_coordtype={RET_COORDTYPE}`. `page_size` caps at 20; dining uses `tag=美食`; radius, terms, and count come from the project requirements.
- Tested 2026-11-08: geocoding, nearby search, reverse geocoding, and the lightweight walking route all returned `status` `0`.

##### Performance

- One geocode plus one reverse geocode per place, in that order (the reverse call depends on the geocode coordinates).

##### Pitfalls

- Baidu returns BD-09 by default; `ret_coordtype=gcj02ll` returns GCJ-02. The `location` parameter order is latitude,longitude — the opposite of AMap.
- Search results are GCJ-02; convert to WGS-84 locally before Open-Meteo. Baidu cannot output WGS-84 (the coordinate converter returns `status` `4` for WGS-84 targets).
- The key goes only in the query string and stays on the server. The browser opens `https://api.map.baidu.com/marker?location={lat},{lng}&title={name}&content={text}&output=html&coord_type=gcj02&src=webapp.{company}.{app}`; `coord_type=gcj02` must be sent or the pin is offset; `src` is required.
- When local time is needed, map the reverse geocode's `adcode` to an IANA zone: `65`→`Asia/Urumqi`, `81`→`Asia/Hong_Kong`, `82`→`Asia/Macau`, `71`→`Asia/Taipei`, any other success→`Asia/Shanghai`. The `adcode` is county-level; the two-digit province prefix is enough. Covering Hong Kong, Macau, and Taiwan requires the global reverse-geocoding service enabled in the console.

#### Google Maps — outside mainland China (incl. Hong Kong, Macau)

##### Environment

```env
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_DIRECTIONS_URL=https://maps.googleapis.com/maps/api/directions/json
GOOGLE_MAPS_PLACES_SEARCH_NEARBY_URL=https://places.googleapis.com/v1/places:searchNearby
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
GOOGLE_MAPS_PLACES_PHOTO_PREFIX=https://places.googleapis.com/v1
GOOGLE_MAPS_FIELD_MASK=places.displayName,places.formattedAddress,places.location,places.photos
```

##### Request

- Geocoding: `GET ${GOOGLE_MAPS_GEOCODE_URL}?address={address}&key=${GOOGLE_MAPS_API_KEY}`. Success is `status` `OK`. Coordinates are WGS-84, ordered latitude,longitude — the opposite of AMap. Tested 2026-10-13: `OK`.
- Directions: `GET ${GOOGLE_MAPS_DIRECTIONS_URL}?origin={lat},{lng}&destination={lat},{lng}&mode=walking&key=${GOOGLE_MAPS_API_KEY}`. Tested 2026-10-13: `OK`.
- Nearby: `POST` searchNearby with a `locationRestriction.circle` body; text search: `POST` searchText with `textQuery` plus `locationBias.circle`. Headers: `Content-Type: application/json`, `X-Goog-Api-Key`, `X-Goog-FieldMask: ${GOOGLE_MAPS_FIELD_MASK}`. Omitting `circle` returns 400. Tested 2026-10-13: both bodies returned HTTP 200 with `places` in the response:

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
  "textQuery": "cantonese",
  "locationBias": {
    "circle": {
      "center": { "latitude": 22.3193, "longitude": 114.1694 },
      "radius": 3000.0
    }
  }
}
```

- Place photo: `GET ${GOOGLE_MAPS_PLACES_PHOTO_PREFIX}/{photo.name}/media?maxHeightPx=800`, header `X-Goog-Api-Key`. `photo.name` is `places/{placeId}/photos/{photoId}`; `maxHeightPx` or `maxWidthPx` (1–4800) is required. The response redirects to the image by default; the server follows the redirect, fetches the bytes, and hands them to the browser; the client `img` points at your own route, without the key. Tested 2026-10-13: following the redirect yields `image/jpeg`.

##### Performance

- Geocode the same place once; later place searches reuse those coordinates.

##### Pitfalls

- Key placement: Geocoding and Directions take the key in the query parameter `key`; Places and place photos take it in the header `X-Goog-Api-Key`. Do not restrict this key by HTTP referrer (server requests carry no Referer and get `REQUEST_DENIED`); use an IP restriction for the key, which can stay unset during local development; enable only the APIs you actually call in Google Cloud.
- Geocoding, Directions, and Places use different hosts. Do not use `https://maps.googleapis.com` as one base URL.
- Mainland China is the `country` short code in the geocode result: `CN` is mainland China; `HK`, `MO`, `TW`, and any other country are not. Reverse geocode a browser location once; do not use a lat/lng bounding box.
- The browser opens `https://www.google.com/maps/dir/?api=1&destination=latitude,longitude`. Do not load the JS API.

### Weather

#### Environment

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

#### Request

`GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m&timezone=auto`. No key; coordinates are WGS-84. Tested 2026-10-13: the response `current` contains all four fields. The `current` fields and `timezone` (`auto` or the project's IANA id) follow that project's requirements; the minimum is `temperature_2m` and `weather_code`.

#### Performance

- Send one forecast with the coordinates already in hand; do not locate the user again for it.

#### Pitfalls

- `weather_code` is a WMO code; map it to copy in the target language before display. Never show the raw code or a description in the wrong language.

### Zodiac MCP

#### Environment

```env
# Streamable HTTP MCP, no key
PANDORIUM_MCP_URL=https://pandorium.app/mcp
```

#### Request

Wire this only when the project requirements name a zodiac service. `POST $PANDORIUM_MCP_URL` with headers `Content-Type: application/json` and `Accept: application/json, text/event-stream`. Tested 2026-10-13: the `tools/call` below returned SSE; the `data:` line after `event: message` is JSON-RPC and `result.content` is text; `tools/list` requires `birthData` in its `inputSchema`, with `location` requiring `latitude`, `longitude`, `timezone`, and `city`.

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
          "city": "Shanghai"
        }
      }
    }
  }
}
```

- On success, parse the SSE `data:` line: `result.content` is an array and the body is `result.content[0].text`. No `content` is failure.
- `get_current_transits` uses the same `birthData`; change `params.name` to that tool.
- With no birth time, `time` is fixed at `12:00`.

#### Performance

- Send one `calculate_natal_chart` with the birth-city coordinates already resolved.
- Whether it runs in parallel with weather, and how many calls per profile per day, come from that project's requirements.

#### Pitfalls

- Never send only the birth date: a `location` missing `city` or the IANA `timezone` is rejected by the schema.
- A 429 body is `{"success":false,"error":"Too many requests","retryAfter":…}`, not JSON-RPC; show `retryAfter` and never retry a 429 automatically.
- Tests use fixtures and never hit the live MCP; leave at least 30 seconds between smoke calls.

### Pokémon TCG catalog

#### Environment

```env
# GET {TCGDEX_BASE_URL}/{lang}/cards/{id}, no key
# lang: zh-cn, zh-tw, en; e.g. /zh-cn/cards/SV7-001
TCGDEX_BASE_URL=https://api.tcgdex.net/v2
```

#### Request

- Wire this only when the project requirements name a card catalog. `GET {TCGDEX_BASE_URL}/{lang}/cards/{id}`.
- Tested 2026-10-13: `GET /v2/zh-cn/cards/SV7-001` and `GET /v2/en/cards/base1-4` returned 200; `zh` in the path returns `language-invalid`.
- Tested 2026-11-13 (mainland China network): `api.tcgdex.net` intermittently went silent for about 15 seconds; the image host `assets.tcgdex.net` is a single OVH server with no CDN — even a 404 took 8s TTFB, and large images timed out past 20 seconds.

#### Performance

- One GET per card id; do not request the same card again.
- Card images and metadata go through the app backend as a proxy with a permanent cache (card images never change); the front end never calls `assets.tcgdex.net` directly. Lists use thumbnails (`low`); only card details use the large image (`high`).

#### Pitfalls

- `lang` is only `zh-cn`, `zh-tw`, or `en`; use only the documented paths and fields.

### Recipe MCP

Three sources serve by cuisine: Chinese from HowToCook (local stdio MCP), Western and other from RecipeWiki (remote MCP) and TheMealDB (REST, not an MCP). None needs a key. When results from several sources are merged, label each item with its source.

#### HowToCook — local stdio MCP

##### Environment

No key. The server runs as a local child process:

```bash
npx -y howtocook-mcp
```

stdio transport: send `initialize` first, then `notifications/initialized`, then `tools/list` and `tools/call`; one JSON-RPC message per line.

##### Request

Tested 2026-11-13 (version 0.2.2, protocol 2024-11-05, loads 322 recipes at startup). Tool names carry the prefix `mcp_howtocook_`:

- `getAllRecipes`: no real parameter (the schema requires a placeholder `no_param`)
- `getRecipesByCategory`: `category` required, enum: 水产, 早餐, 调料, 甜品, 饮品, 荤菜, 半成品加工, 汤, 主食, 素菜
- `recommendMeals`: `peopleCount` required (1–10), optional `allergies`, `avoidItems`
- `whatToEat`: `peopleCount` required
- `getRecipeById`: fetches one dish by id

The return is in `result.content[0].text` as a JSON array; each element has `id` (like `dishes-aquatic-咖喱炒蟹`), `name`, and `description` (the Markdown body).

##### Performance

- Keep the child process alive and reuse it; do not restart it per call.
- The bottleneck is not latency (data is local; every call measured under 0.01s on 2026-11-13) but the context cost of response size. Measured sizes per tool: `getRecipeById` about 3.4KB (one dish, smallest); `whatToEat` about 6.5KB; `getAllRecipes` about 43.5KB (all 322 recipes); `getRecipesByCategory` up to about 40KB for one category (水产); `recommendMeals` about 137KB (a week's meal plan plus shopping list — the largest).
- Prefer `getRecipeById` (fuzzy match by name) and `getRecipesByCategory`. Before pulling a list, ask whether the full corpus or a week plan really belongs in the context; the official README itself warns on `getAllRecipes`: 「慎用这个——上下文太大」(use with caution — too large for the context).

##### Pitfalls

- `getRecipeById` takes a parameter named `query` (recipe name or id, fuzzy match supported); any other parameter name such as `id` or `name` is rejected by the schema (`-32602`).
- Version differences: 0.2.2 bundles the recipe data (322 recipes load at startup); earlier versions fetched `all_recipes.json` from the remote host `weilei.site` at startup and occasionally died on a 10-second connect timeout (official issue #11). Pin the version in production to avoid behavior drift.
- The source text is Chinese. When the target language is not Chinese, translate before display; translation caching and labeling rules come from that project's requirements.

#### RecipeWiki — remote MCP

##### Environment

```env
# Streamable HTTP MCP, no key, no account
RECIPEWIKI_MCP_URL=https://www.recipewiki.com/api/mcp
```

##### Request

`POST $RECIPEWIKI_MCP_URL` with headers `Content-Type: application/json` and `Accept: application/json, text/event-stream`. Tested 2026-11-13 (protocol 2025-03-26, server recipewiki 1.0.0): responses are plain `application/json` JSON-RPC, not SSE; no session header is needed. Tools:

- `search_recipes`: args `query`, `tags`, `cuisine`, `course`, `sort`, `page`, `limit`, all optional. Returns summaries (title, link, photo, timings, servings, allergens) without ingredients or method — call `get_recipe` for those.
- `get_recipe`: `slug` required. Returns a schema.org Recipe JSON string in `result.content[0].text`, with `recipeIngredient`, `recipeInstructions`, `prepTime`, `cookTime`, `totalTime`, `recipeYield`, and more.
- `list_tags`: optional `group`.
- `list_allergens`: no args.

##### Performance

- List and detail are two calls: `search_recipes` first, then `get_recipe` for the chosen `slug`.

##### Pitfalls

- `query` ignores `tags`, `cuisine`, and `course` (stated in the tool description); do not combine them and expect both to apply.
- Allergens are rolled up through sub-recipes; read the `list_allergens` note before declaring a dish safe.

#### TheMealDB — REST (not an MCP)

##### Environment

```env
# The 1 in the path is the official public test key; no application needed
THEMEALDB_BASE_URL=https://www.themealdb.com/api/json/v1/1
```

##### Request

Tested 2026-11-13; all returned HTTP 200:

- Search by name: `GET {THEMEALDB_BASE_URL}/search.php?s={name}`
- Detail by id: `GET {THEMEALDB_BASE_URL}/lookup.php?i={idMeal}`
- Random dish: `GET {THEMEALDB_BASE_URL}/random.php`
- Category list: `GET {THEMEALDB_BASE_URL}/list.php?c=list`
- Filter by category: `GET {THEMEALDB_BASE_URL}/filter.php?c={category}`

The response is `{"meals":[…]}` with `idMeal`, `strMeal`, `strCategory`, `strArea`, `strInstructions`, and `strMealThumb`.

##### Performance

- One fetch per request; do not request the same recipe again.

##### Pitfalls

- The source text is English. When the target language is not English, translate before display; translation caching and labeling rules come from that project's requirements.
- `filter.php` returns a short list only (no steps or ingredients); use `lookup.php` for details.

## 4. Install

### npm mirror

Install dependencies by current location. When `package.json` is missing, create the app from the project requirements first, then install; when a lockfile exists, use `npm ci` and never rewrite it with `npm install`.

Mainland China, Hong Kong, and Macau:

```bash
# only to generate a lockfile when none exists
npm install --registry=https://registry.npmmirror.com
# when a lockfile exists
npm ci --registry=https://registry.npmmirror.com
```

Other regions:

```bash
# only to generate a lockfile when none exists
npm install
# when a lockfile exists
npm ci
```

- With a lockfile present, use only `npm ci`; rewriting it with `npm install` turns `resolved` entries into mirror URLs. `registry.npmjs.org` inside a lockfile means "the currently configured registry"; `npm ci --registry=...` downloads from the mirror without touching the lockfile.
- Never run `npm config set registry`; never commit a project `.npmrc`.
- On an `edgesOut` null pointer, rerun `npm ci` with `--legacy-peer-deps`; the same applies to other peer-dependency failures. Do not change versions in the lockfile and do not change `@types/node` to make an install pass.
- If `npm config get registry` is not `https://registry.npmjs.org/`, clear the user-level setting with `npm config delete registry` and run `npm ci` again.

### Secrets

- Secrets live in the repo-root `.secrets`. A person places that file first; the real values live only there.
- At the start, copy only the variable names this project uses from the repo-root `.secrets` into that project's `.env.local`; skip the variables `.secrets` has but this project does not use. If a needed variable is missing or empty, stop.
- At runtime only `.env.local` is read. No `NEXT_PUBLIC_*` for secrets. Lock the model id with an environment variable; do not rely on the provider default.
- Never put a secret in a reply, a log, or a browser response. Do not commit `.secrets` or `.env.local`.

## 5. Call order

Send independent calls together. Show a block when that block is ready. Follow an order the requirements name. A failure explains only that block. Do not retry automatically.

## 6. Tests

This section covers tooling and how tests run. Business cases, fixtures, and pass criteria live in that project's test files, not in the requirements and not here.

- Unit and component tests use Vitest. When a page must render, use `jsdom` and Testing Library.
- API and integration tests use the same Vitest suite and hit local routes. External map, chat, and image calls use fixtures; no network.
- Browser journeys named by test files use Playwright. Without a named journey, no end-to-end run.
- At least one critical user journey is covered by a real browser end-to-end test.
- Test data goes to a temp directory (settable via `DATA_DIR`), not the project's `data/`. Secrets stay out of test code and logs.
