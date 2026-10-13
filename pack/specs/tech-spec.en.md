# Tech spec

This document defines the stack, third-party capabilities, and cross-cutting implementation rules. Product behavior lives in each project's requirements doc; MyDay-specific rules are in section 5. Chinese version: [`tech-spec.zh.md`](./tech-spec.zh.md).

## 1. Third-party capabilities

Wire only what the project needs. There is no reviews provider. Failure semantics, performance, and secrets are in section 3.

### Chat

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

Key pitfalls:

- Pass `QUANZIL_API_KEY` and `QUANZIL_BASE_URL` into the client constructor. Do not set `OPENAI_API_KEY` or `OPENAI_BASE_URL`; the SDK reads those by default and calls `api.openai.com`
- Use `max_completion_tokens`, not `max_tokens`

### Image generation

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

Key pitfalls:

- Pass the key as the `key` query parameter, not a header. REST on the server only; prefer AMap URI H5 in the browser — do not ship a JS API key to the client (see MyDay appendix)
- Geocoding is `/v3`. Text search, nearby search, and walking directions are `/v5`. Do not set one `AMAP_BASE_URL=/v3` for every call
- Browser geolocation and Open-Meteo use WGS-84; AMap REST uses GCJ-02 — convert coordinates

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

Key pitfalls:

- One server `GOOGLE_MAPS_API_KEY`. No `NEXT_PUBLIC_*`. Geocoding and Directions send it as the `key` query parameter. Places and place photos send it as `X-Goog-Api-Key`
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
npm ci --registry=https://registry.npmmirror.com
```

Key pitfalls:

- When a lockfile exists, use `npm ci`. `npm install --registry=...` rewrites `resolved` to the mirror
- `registry.npmjs.org` in the lockfile means the configured registry. `npm ci --registry=https://registry.npmmirror.com` downloads from the mirror and does not edit the lockfile
- Do not run `npm config set registry`. Do not commit a project `.npmrc`
- On `edgesOut`, rerun `npm ci --registry=https://registry.npmmirror.com --legacy-peer-deps`. Do not change pinned versions

#### Hong Kong and Macau

Same as **Mainland China** (npmmirror).

#### Other regions

```bash
npm ci
```

When a lockfile exists, use `npm ci`, not `npm install`. Do not run `npm config set registry` and do not commit a project `.npmrc`. If `npm config get registry` is not `https://registry.npmjs.org/`, run `npm config delete registry`, then `npm ci`.

### Weather

**Same in all regions.** Open-Meteo, no API key.

```env
# GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m&timezone=auto
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

`latitude` and `longitude` are WGS-84. On 2026-10-13 this query returned `current` with `temperature_2m`, `relative_humidity_2m`, `weather_code`, and `wind_speed_10m`.

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

`get_current_transits` uses the same `birthData` and changes `params.name`. Parse the SSE `data:` line: `result.content` is an array, and the chart is the JSON string at `result.content[0].text`. No `content` means failure. When the profile has no birth time, `time` is `12:00` (MyDay: section 5).

Key pitfalls:

- Date of birth alone is not enough. A missing `city` or IANA `timezone` fails the schema
- 429 body is `{"success":false,"error":"Too many requests","retryAfter":…}`, not JSON-RPC — show `retryAfter`; never auto-retry 429
- Use fixtures in unit/integration/CI; no live MCP on the network; ≥30s between live smoke calls
- Parallel fetch and once-per-day limits live in the project appendix (MyDay: section 5)

### Pokémon TCG catalog

**Same in all regions.** TCGdex, HTTPS GET, no key.

```env
# GET {TCGDEX_BASE_URL}/{lang}/cards/{id}
# lang: zh-cn, zh-tw, or en. Example: /zh-cn/cards/SV7-001
TCGDEX_BASE_URL=https://api.tcgdex.net/v2
```

On 2026-10-13 `GET /v2/zh-cn/cards/SV7-001` and `GET /v2/en/cards/base1-4` returned 200. The path segment `zh` returns `language-invalid`.

Key pitfalls:

- `lang` is `zh-cn`, `zh-tw`, or `en`. Use documented paths and fields only

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

Rules: minimal dependencies; prefer the platform and stdlib; **no database** (local JSON under `data/` only unless a project spec says otherwise). npm registry: section 1.

## 3. Cross-cutting contracts

### 3.1 Failure semantics

- On third-party failure, return the error to the caller for display and retry — never silent empty success, empty image, or empty list.
- Local fallback only when upstream fails, or the component is still empty after the user-visible wait path.

### 3.2 Performance (all components)

- Soft tip: after ~**10s**, show "more time is needed" while work continues — never the word "Timeout".
- Target: each component ready within **≤20s**.
- Server outbound timeout ~**25s** — do not add a separate 10s hard abort that kills an in-flight request.
- Retry: at most one explicit user-triggered retry per attempt path; no auto-retry loops inside 20s.
- Loading: fetch independent components in parallel; render each piece as it completes; show per-component progress.

### 3.3 Secrets

- Runtime keys only in `.env.local` (gitignored; never commit). Env var names are listed below.
- No `NEXT_PUBLIC_*` exposure of map or LLM keys. Lock model ids in env; do not rely on a provider default model.

### 3.4 Quality bar

- Unit tests for business logic; 100% coverage of the changed critical path
- Integration tests across API / data-boundary paths (fixtures; no outbound network by default)
- At least one critical user journey covered by E2E in a real browser
- Tests use a temporary data directory
- No database: persistence is local JSON under `data/`

### 3.5 Copy and i18n

Whether UI copy uses i18n is decided by that project's requirements doc; if i18n is not required, hard-coded strings are allowed.

### 3.6 Pandorium and weather in parallel

Rate limits, `birthData`, and test rules for the zodiac MCP are in section 1 (**Zodiac MCP**). Component rule: fetch weather and zodiac in parallel; at most one natal chart and one transit call per profile per day.

## 4. Environment variable index

Each capability in section 1 includes an example `.env.local` block. Enable only what the project uses. Optional:

```env
DATA_DIR=
```

## 5. Project appendix: MyDay

Product behavior is in [`../projects/myday/req.en.md`](../projects/myday/req.en.md). Implementation rules follow below. Chat, image, and maps: section 1 (chat and image both via Quanzil on one key: chat `gpt-6-astra`, image `gemini-3.1-flash-lite-image`; mainland AMap, everywhere else Google Maps).

### Dependencies

Do not add: Prisma / any database, an MCP SDK, the Maps JavaScript loader as a client key, or `NEXT_PUBLIC_*` secrets. Image downscaling uses `sharp` bundled with Next. Coordinate conversion, WMO mapping, sun-sign calculation, and MCP JSON-RPC use the stdlib and `fetch`.

### Storage and routes

| Path | Responsibility |
| --- | --- |
| `data/profiles.json` | Profile array (id, name, gender, date of birth, height, weight, occupation, `photoPath` filename or empty) |
| `data/uploads/` | User photos and downloaded outfit images |
| Tests | `DATA_DIR` points at a temp dir; default `data/` |

`photoPath` stores a filename only. Read it from `data/uploads/` via `uploadsDir()`, not `cwd + photoPath`. A read failure must error. When a profile photo exists, pass that file as `generateContent` `inlineData` for every region. Never drop it and call again with text only.

Route Handlers must not return secrets to the client. Day-plan chunks (weather, zodiac, outfit text, outfit image, restaurant list) are independent and parallel; a failure affects only its own chunk.

### Google Maps (Hong Kong, Macau, and other regions)

- Nearby search stays within 3 km. Stop at the first non-empty result. Do not widen the radius.
- Empty results return an error or copy that tells the user to change cuisine or location — never a blank list.
- Place photos go through a server photo proxy. Do not put the Maps key in client `img` URLs.
- Invalid location: coordinates near `35.86, 104.20` (about 0.05° tolerance) or `accuracy > 8000m`. Do not enter the day plan with an invalid fix.

Mainland China MyDay uses AMap instead: server REST only, URI H5 for the map overlay (no JS API key in the browser), WGS-84 for the browser and Open-Meteo, GCJ-02 for AMap. See the Chinese spec appendix for the retrieval ladder and cuisine keywords.

### Weather and zodiac

- Open-Meteo: `latitude` and `longitude` (WGS-84) plus `timezone=auto`. Map `weather_code` to display-language copy before showing it.
- Call Pandorium with `fetch` and parse SSE `data:` lines. Do not add an MCP SDK. `birthData` shape:

```json
{
  "date": "1990-03-15",
  "time": "12:00",
  "location": {
    "latitude": 31.2304,
    "longitude": 121.4737,
    "timezone": "Asia/Shanghai",
    "city": "Shanghai"
  }
}
```

- When the profile has no birth time, `time` is `12:00`. Do not call the MCP with an invalid location. Compute sun sign and age locally.
- Fetch weather and zodiac in parallel. Do not put `weather` in the zodiac request `useEffect` dependencies.
- At most one natal chart and one transit call per profile per day.

### Tests

Critical-path unit coverage includes: invalid-location detection, WGS-84↔GCJ-02 where AMap is used, WMO mapping, sun sign, and "photo present means reference image attached". E2E: profile → plan → day-plan chunks render.

### i18n

Supported locales and copy rules are in `req.en.md`.
