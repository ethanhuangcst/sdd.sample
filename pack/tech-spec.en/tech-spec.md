# Tech spec

This document defines the stack, third-party capabilities, and cross-cutting implementation rules. Product behavior lives in each project's `specs/req-spec.md` (seed from `pack/projects/<locale>/req-spec.md`). Chinese version: [`../tech-spec.cn/tech-spec.md`](../tech-spec.cn/tech-spec.md).

## 1. Third-party capabilities

Wire only what the project needs. There is no reviews provider. Failure semantics, performance, and secrets are in section 3.

### Chat LLM

**Same in all regions.** Provider: **Quanzil** (OpenAI-compatible chat completions). One `QUANZIL_API_KEY` shared with image generation. Chat uses the `openai` SDK. The image request shape is in the next section.

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6-astra
```

Request. On 2026-10-13 this call returned a non-empty `choices[0].message.content`.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.QUANZIL_API_KEY,
  baseURL: process.env.QUANZIL_BASE_URL,
});

const completion = await client.chat.completions.create({
  model: process.env.QUANZIL_CHAT_MODEL,
  messages: [{ role: "user", content: "task summary" }],
  max_completion_tokens: 1024,
});

const text = completion.choices[0]?.message?.content ?? "";
```

Empty `text` is failure.

Performance:

- One short chat. On the same user wait, do not place chat ahead of image generation.

Key pitfalls:

- Pass `QUANZIL_API_KEY` and `QUANZIL_BASE_URL` into the client constructor. Do not set `OPENAI_API_KEY` or `OPENAI_BASE_URL`; the SDK reads those by default and calls `api.openai.com`
- Use `max_completion_tokens`, not `max_tokens`

### Image generation LLM

**Same in all regions.** Same `QUANZIL_API_KEY` as Chat. Call Gemini `generateContent`, not the `openai` SDK images methods.

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

Request: `POST $QUANZIL_IMAGE_URL` with `Authorization: Bearer <QUANZIL_API_KEY>` and `Content-Type: application/json`. On 2026-10-13 a text-only call and a 64×64 JPEG `inlineData` both returned HTTP 200 with `inlineData.mimeType` `image/jpeg`. A 1×1 JPEG returned 400 `Unable to process input image`. The reference photo must be a decodable image.

```json
{
  "contents": [{
    "role": "user",
    "parts": [
      { "text": "task summary" },
      { "inlineData": { "mimeType": "image/jpeg", "data": "<base64>" } }
    ]
  }],
  "generationConfig": { "responseModalities": ["IMAGE"] }
}
```

With no user photo, `parts` contains only `text`. With a photo, include `inlineData` after server downscale (long edge ~1024px, JPEG/WebP). A read failure must error. Do not delete `inlineData` and call again.

Read the image from `candidates[0].content.parts[].inlineData.data` (base64). `mimeType` is on that same object. No `inlineData` means failure.

Performance:

- Send the call when the photo and the short prompt are ready. On the same wait, do not place a chat call ahead of the image.
- Convert the reference to JPEG or WebP and limit the long edge to about 1024px, then send it. Do not shrink it further to save time. The output stays about 1024×1024, and a smaller reference does not shorten generation.
- Do not set `imageConfig.imageSize`. `512` returns 400.
- Wait for the image on its own. Do not cut it off with the short wait used for maps or weather.

Key pitfalls:

- `generationConfig.responseModalities` must be `["IMAGE"]`. Without it the call often returns text only
- The model id must appear in `GET /v1/models` for this key
- When text is already available, do not serialise text LLM and image on one critical path
- Short, structured prompts — task summary only

### Maps

Map keys are server-only. No `NEXT_PUBLIC_*` map keys.

#### Mainland China

- Provider: **AMap** (Web-service key, server only)
- Uses: geocoding, place search, nearby POI, routing

Example `.env.local`:

```env
AMAP_API_KEY=your-web-service-key
AMAP_HOST=https://restapi.amap.com
# GET {AMAP_HOST}/v3/geocode/geo?address={address}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/text?keywords={keywords}&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/place/around?location={lng},{lat}&radius=3000&key=${AMAP_API_KEY}
# GET {AMAP_HOST}/v5/direction/walking?origin={lng},{lat}&destination={lng},{lat}&key=${AMAP_API_KEY}
```

`location`, `origin`, and `destination` are `longitude,latitude`. Success is HTTP 200 with `status` `1` and `infocode` `10000`. On 2026-10-13 all four paths returned `infocode` `10000` with these query parameters.

Performance:

- Geocode a place once. Weather and restaurants use the current place. The natal chart uses the birth city. When those are different places, geocode each once.

Key pitfalls:

- Pass the key as the `key` query parameter, not a header. REST on the server only. Open the map with `https://uri.amap.com/marker?position=longitude,latitude`. Do not load the JS API, and do not ship a JS API key to the client
- Geocoding is `/v3`. Text search, nearby search, and walking directions are `/v5`. Do not set one `AMAP_BASE_URL=/v3` for every call
- Browser geolocation and Open-Meteo use WGS-84. AMap REST uses GCJ-02. Convert between WGS-84 and GCJ-02 in project code, and do not add a dependency for it. Convert AMap coordinates to WGS-84 before deciding whether the place is in mainland China, and before the natal chart and weather. Keep the original GCJ-02 coordinates for restaurant search. A typed city from AMap is already GCJ-02
- Decide mainland China from the geocode admin result, not from a latitude and longitude box. Reverse-geocode browser coordinates with `GET /v3/geocode/regeo?location={longitude},{latitude}` first. An `adcode` starting with 71, 81, or 82 is Taiwan, Hong Kong, or Macau, and is not mainland China. An empty forward-geocode result is a failed resolve

##### Nearby dining search

Use this when a project searches nearby restaurants by cuisine. Whether the radius may grow is decided by that project's requirements.

- Call `GET /v5/place/around`, not `/v5/place/text`
- `location` is GCJ-02, ordered longitude,latitude. `radius` is meters
- Dining is `types=050000`
- The search word is the text the user entered
- Stop at the first non-empty result: search word plus `types=050000`, then the search word only, then `types=050000` only
- When the requirements do not say the range may grow, do not increase `radius` and search again

#### Hong Kong and Macau

- Provider: **Google Maps Platform**

Example `.env.local`:

```env
GOOGLE_MAPS_API_KEY=your-server-key
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_DIRECTIONS_URL=https://maps.googleapis.com/maps/api/directions/json
GOOGLE_MAPS_PLACES_SEARCH_NEARBY_URL=https://places.googleapis.com/v1/places:searchNearby
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
GOOGLE_MAPS_PLACES_PHOTO_PREFIX=https://places.googleapis.com/v1
GOOGLE_MAPS_FIELD_MASK=places.displayName,places.formattedAddress,places.location,places.photos
```

Performance:

- Geocode a place once. Reuse those coordinates for later place search. Do not geocode the same place again.

Key pitfalls:

- One server `GOOGLE_MAPS_API_KEY`. No `NEXT_PUBLIC_*`. Geocoding and Directions send it as the `key` query parameter. Places and place photos send it as `X-Goog-Api-Key`. Open the map with `https://www.google.com/maps/dir/?api=1&destination=latitude,longitude`. Do not load the JS API
- Decide mainland China from the geocode `country` short name. `CN` is mainland China. `HK`, `MO`, `TW`, and any other country are not. Reverse-geocode browser coordinates once. Do not use a latitude and longitude box
- Geocoding: `GET ${GOOGLE_MAPS_GEOCODE_URL}?address={address}&key=${GOOGLE_MAPS_API_KEY}`. Directions: `GET ${GOOGLE_MAPS_DIRECTIONS_URL}?origin={lat},{lng}&destination={lat},{lng}&mode=walking&key=${GOOGLE_MAPS_API_KEY}`. That order is `latitude,longitude`, the reverse of AMap. Success is `status` `OK`. On 2026-10-13 both calls returned `OK`
- Places: `POST` the search URL with `Content-Type: application/json`, `X-Goog-Api-Key`, and `X-Goog-FieldMask` from `GOOGLE_MAPS_FIELD_MASK`. `searchNearby` body:

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

`searchText` body:

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

`radius` is meters. Omitting `circle` returns 400. On 2026-10-13 both bodies returned HTTP 200 with a `places` array.
- Place photo: `GET ${GOOGLE_MAPS_PLACES_PHOTO_PREFIX}/{photo.name}/media?maxHeightPx=800` with header `X-Goog-Api-Key`. `photo.name` is `places/{placeId}/photos/{photoId}`. `maxHeightPx` or `maxWidthPx` (1–4800) is required. The default response redirects to the image; follow it on the server and return the bytes. On 2026-10-13 that request followed the redirect to `image/jpeg`. The client `img` URL points at your route and has no key
- Do not use `https://maps.googleapis.com` as one base URL. Restrict the key by IP, not by HTTP referrer
- Enable only the APIs you call

#### Other regions

Same as **Hong Kong and Macau** (Google Maps and the env block and pitfalls above).

### npm registry

#### Mainland China

```bash
# No lockfile yet: this command only creates it
npm install --registry=https://registry.npmmirror.com
# Lockfile present
npm ci --registry=https://registry.npmmirror.com
```

Key pitfalls:

- When a lockfile exists, use only `npm ci`. Do not use `npm install` to rewrite it. That rewrites `resolved` to the mirror
- `registry.npmjs.org` in the lockfile means the configured registry. `npm ci --registry=https://registry.npmmirror.com` downloads from the mirror and does not edit the lockfile
- Do not run `npm config set registry`. Do not commit a project `.npmrc`
- On `edgesOut`, rerun `npm ci --registry=https://registry.npmmirror.com --legacy-peer-deps`. Do not change pinned versions

#### Hong Kong and Macau

Same as **Mainland China** (npmmirror).

#### Other regions

```bash
# No lockfile yet: this command only creates it
npm install
# Lockfile present
npm ci
```

When a lockfile exists, use only `npm ci`. Do not use `npm install` to refresh dependencies. Do not run `npm config set registry` and do not commit a project `.npmrc`. If `npm config get registry` is not `https://registry.npmjs.org/`, run `npm config delete registry`, then `npm ci`.

### Weather

**Same in all regions.** Open-Meteo, no API key.

```env
# GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m&timezone=auto
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

`latitude` and `longitude` are WGS-84. On 2026-10-13 this query returned `current` with `temperature_2m`, `relative_humidity_2m`, `weather_code`, and `wind_speed_10m`.

Performance:

- Send one forecast with the WGS-84 coordinates already in hand. Do not locate the user again for that forecast.

Key pitfalls:

- `weather_code` is a WMO code — map it to display-language copy before showing it

### Zodiac MCP

**Same in all regions.** Pandorium, Streamable HTTP MCP, no key.

```env
PANDORIUM_MCP_URL=https://pandorium.app/mcp
```

Request: `POST $PANDORIUM_MCP_URL` with `Content-Type: application/json` and `Accept: application/json, text/event-stream`. On 2026-10-13 the `tools/call` below returned SSE: an `event: message` line, then a `data:` line of JSON-RPC whose `result.content` is text. `tools/list` `inputSchema` requires `birthData`, and `location` requires `latitude`, `longitude`, `timezone`, and `city`.

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

`get_current_transits` uses the same `birthData` and changes `params.name`. Parse the SSE `data:` line: `result.content` is an array, and the chart is the JSON string at `result.content[0].text`. No `content` means failure. When the profile has no birth time, `time` is `12:00`.

Performance:

- Send one `calculate_natal_chart` with the birth-city coordinates already resolved. When the profile has no birth time, set `time` to `12:00` directly.

Key pitfalls:

- Date of birth alone is not enough. A missing `city` or IANA `timezone` fails the schema
- 429 body is `{"success":false,"error":"Too many requests","retryAfter":…}`, not JSON-RPC — show `retryAfter`; never auto-retry 429
- Use fixtures in unit/integration/CI; no live MCP on the network; ≥30s between live smoke calls
- Whether weather and zodiac run together, and how many calls one profile may make in a day, is set by that project's requirements doc

### Pokémon TCG catalog

**Same in all regions.** TCGdex, HTTPS GET, no key.

```env
# GET {TCGDEX_BASE_URL}/{lang}/cards/{id}
# lang: zh-cn, zh-tw, or en. Example: /zh-cn/cards/SV7-001
TCGDEX_BASE_URL=https://api.tcgdex.net/v2
```

On 2026-10-13 `GET /v2/zh-cn/cards/SV7-001` and `GET /v2/en/cards/base1-4` returned 200. The path segment `zh` returns `language-invalid`.

Performance:

- Send one GET for the card id. Do not request the same card again.

Key pitfalls:

- `lang` is `zh-cn`, `zh-tw`, or `en`. Use documented paths and fields only

## Design principles

- Dependencies and persistence are in section 3.3.
- Secrets are in section 3.4. Plaintext secrets live in `specs/.secrets`. Runtime reads `.env.local` only. Do not commit either file. Do not put secrets in replies, logs, or client responses.

## 2. Stack

Majors below (reference date 2026-10). Pin exact patch versions in your project's root `package.json` / lockfile.

| Category | Tech | Major |
| --- | --- | --- |
| Framework | Next.js (App Router) | 16.4 |
| UI / language | React · TypeScript | 19.3 · 7.0 (native port; confirm editor and build locally) |
| Styles | Tailwind CSS | 4.3 |
| State / forms | React Query · Zustand · RHF + Zod + `@hookform/resolvers` | 5.x · 5.x · 7.x + 4.x + 5.x |
| Storage | Local JSON (`data/`) — no DB | — |
| AI SDK | `openai` SDK for chat only (Quanzil chat completions). Image is Gemini `generateContent` — section 1 | 7.x (6→7 has breaking changes; walk the official migration guide) |
| Test | Vitest · RTL · Playwright | 5.x · 16.x · 1.64 |

Dependencies and persistence are in section 3.3. npm registry: section 1. TypeScript 7.0 in the table is the target version. When `create-next-app` creates the project, use the TypeScript that command installs. Do not change it to 7.0 while creating the project. Packages named in the install list use the versions in this table. Do not install packages the list omits. See section 3.3.

## 3. Cross-cutting contracts

### 3.1 Failure semantics

- On third-party failure, return the error so the UI can explain why. Never silent empty success, empty image, or empty list.
- Whether the user may try that call once more is decided by the project requirements. When the requirements do not mention retry, the user may retry once by hand. Do not retry in a loop.
- Local fallback only when upstream fails, or the component is still empty after the user-visible wait. Do not switch to a fallback while the wait is still running.

### 3.2 Performance (all components)

- Send independent calls together.
- Show a block when that block is ready. When the requirements say to fetch first and show the result on the next page, follow the requirements.
- When a call has no result, explain why under section 3.1 and end that wait. How to do the call is in that capability's Performance section.

### 3.3 Minimal dependencies

- Do not add a framework the project does not use. Prefer the platform and the standard library.
- When the install list in the project requirements is shorter than the table in section 2, follow the requirements.
- Persist with local files only. Profiles and uploads live under `data/`. No database and no ORM.

### 3.4 Secrets

- Runtime keys only in `.env.local` (gitignored; never commit). Env var names are listed below.
- No `NEXT_PUBLIC_*` exposure of map or LLM keys. Lock model ids in env; do not rely on a provider default model.

### 3.5 Quality bar

- Unit tests for business logic; 100% coverage of the changed critical path
- Integration tests across API / data-boundary paths (fixtures; no outbound network by default)
- At least one critical user journey covered by E2E in a real browser
- Tests use a temporary data directory
- When the project requirements include a test section, and that section is shorter than this one, follow the requirements.

### 3.6 Copy and i18n

Whether UI copy uses i18n is decided by that project's requirements doc; if i18n is not required, hard-coded strings are allowed.

### 3.7 Pandorium and weather in parallel

Rate limits, `birthData`, and test rules for the zodiac MCP are in section 1 (**Zodiac MCP**). Whether weather and zodiac run together, and how many calls one profile may make in a day, is set by that project's requirements doc.

## 4. Environment variable index

Each capability in section 1 includes an example `.env.local` block. Enable only what the project uses. Optional:

```env
DATA_DIR=
```
