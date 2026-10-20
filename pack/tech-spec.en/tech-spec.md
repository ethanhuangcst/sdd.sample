# Tech spec


## 1. Principles

- This file is the shared tech spec for practice projects. Product behavior lives in that project's requirements.
- Later projects may include a web app, MCP, RAG, and an agent. This file sets the stack, third-party capabilities, and install steps those parts use. Do not wire a capability the project requirements do not name.
- A third-party failure must tell the UI why. No silent empty success, empty image, or empty list. When the requirements do not mention retry, the user may retry once by hand. Do not retry in a loop.
- When the install list or the test section in the project requirements is shorter than this file, follow the requirements.
- Whether user-facing copy uses i18n is decided by that project's requirements. When they do not require it, one language written in the UI is allowed.

## 2. Stack

Majors below (reference date 2026-10). The exact patch follows the project's `package.json` and lockfile. When `create-next-app` creates the project, use the TypeScript that command installs. Do not change it to 7.0 while creating the project.

| Category | Tech | Major |
| --- | --- | --- |
| Framework | Next.js (App Router) | 16.4 |
| UI / language | React, TypeScript | 19.3, 7.0 |
| Styles | Tailwind CSS | 4.3 |
| State / forms | React Query, Zustand, RHF + Zod | 5.x, 5.x, 7.x |
| Storage | Local JSON (`data/`), no database | — |
| AI SDK | `openai` SDK for chat only. Image uses Gemini `generateContent` | 7.x |
| Test | Vitest, RTL, Playwright | 5.x, 16.x, 1.64 |

## 3. Capabilities

### Chat

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6-astra
```

- The provider is Quanzil. The API is OpenAI-compatible chat completions. Use the `openai` SDK.
- Pass the key and base URL above into the client constructor. Do not also set `OPENAI_API_KEY` or `OPENAI_BASE_URL`.
- Use `max_completion_tokens`, not `max_tokens`. An empty `choices[0].message.content` is failure.
- `gpt-6-astra` counts reasoning tokens inside `max_completion_tokens` before it writes `choices[0].message.content`. The word and character limits in the requirement prompts do not set this cap. Set `max_completion_tokens` to 1200 for the status summary, the itinerary, and the outfit text. An empty `choices[0].message.content` is failure, including when `finish_reason` is `length` because reasoning used the whole cap.

### Image

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

- Share `QUANZIL_API_KEY` with chat. `POST` the URL above. Header `Authorization: Bearer <QUANZIL_API_KEY>`.
- The body includes `generationConfig.responseModalities: ["IMAGE"]`. Do not use the `openai` images methods. Do not set `imageConfig.imageSize`.
- With a photo, `parts` include text and `inlineData`. `mimeType` is the photo's own type: `image/png`, `image/jpeg`, `image/webp`, `image/heic`, or `image/heif`. The page-1 photo is small: the user controls the upload, long edge within about 2048px and file within 2MB. The app does not scale images. Wrap the image POST in a 75-second timeout; a timeout fails that block, no retry. A 1×1 image returns 400.
- The result is in `candidates[0].content.parts[].inlineData`. No `inlineData` is failure. A photo that cannot be read is an error. Do not send the request again without the photo.

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

- Geocoding: `GET ${GOOGLE_MAPS_GEOCODE_URL}?address={address}&key=`. Success is `status` `OK`. Coordinates are WGS-84, in the order latitude,longitude.
- Directions: `GET ${GOOGLE_MAPS_DIRECTIONS_URL}?origin={latitude},{longitude}&destination={latitude},{longitude}&mode=walking&key=`.
- Nearby: `POST` searchNearby. The body is `locationRestriction.circle`. Omitting `circle` returns 400.
- Text search: `POST` searchText. The body is `textQuery` plus `locationBias.circle`. Omitting `circle` returns 400.
- Place photo: `GET ${GOOGLE_MAPS_PLACES_PHOTO_PREFIX}/{photo.name}/media?maxHeightPx=800`, header `X-Goog-Api-Key`. `maxHeightPx` or `maxWidthPx` is required.
- Places headers also include `Content-Type: application/json` and `X-Goog-FieldMask`. Geocoding and Places use different hosts. Do not use `https://maps.googleapis.com` as one base URL.
- The key stays on the server. No `NEXT_PUBLIC_*`. Do not restrict this key by HTTP referrer.
- The browser opens `https://www.google.com/maps/dir/?api=1&destination=latitude,longitude`. Do not load the JS API.
- Geocode the same place once.
- Mainland China is the `country` short code in the result. `CN` is mainland China. `HK`, `MO`, `TW`, and any other country are not.
- The radius comes from that project's requirements. When the requirements do not say the range may grow, do not increase the radius and search again.

### Weather

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

- `GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,weather_code`. Coordinates are WGS-84. No key.
- Send one request with the coordinates already in hand. Do not locate the user again for weather.
- `weather_code` is a WMO code. Translate it into the project's language before display. Do not show the raw code, or a description that is not in the project's language.

### Zodiac MCP

```env
PANDORIUM_MCP_URL=https://pandorium.app/mcp
```

- Wire this only when the project requirements name a zodiac service. `POST` that URL. Headers: `Content-Type: application/json` and `Accept: application/json, text/event-stream`.
- The tool name is `calculate_natal_chart`. The argument is `birthData`: `date`, `time`, `location.latitude`, `location.longitude`, `location.timezone`, and `location.city`. When there is no birth time, `time` is `12:00`.
- On success, parse the SSE `data:` line. The body is `result.content[0].text`. No `content` is failure.
- A 429 response is not JSON-RPC. Show `retryAfter`. Do not retry a 429 automatically.
- Tests use fixtures and do not call the network. Leave at least 30 seconds between smoke calls.

### Pokémon

```env
TCGDEX_BASE_URL=https://api.tcgdex.net/v2
```

- Wire this only when the project requirements name a card catalog. `GET {TCGDEX_BASE_URL}/{lang}/cards/{id}`. No key.
- `lang` is `zh-cn`, `zh-tw`, or `en`. Send one request for the card id. Do not request the same card again.

## 4. Install

- When `package.json` is not there yet, create the app from the project requirements, then install. When a lockfile exists, use `npm ci`. Do not use `npm install` to rewrite the lockfile.
- Mainland China, Hong Kong, and Macau: `npm install --registry=https://registry.npmmirror.com`. When a lockfile exists, use `npm ci --registry=https://registry.npmmirror.com`.
- Other regions: `npm install` or `npm ci`. Do not set a mirror.
- Do not run `npm config set registry`. Do not commit a project `.npmrc`.
- When a peer dependency fails, rerun that failed command with `--legacy-peer-deps`. Do not change `@types/node` to make the install pass.
- Secrets are in `specs/.secrets`. A person places that file first. Copy only the variable names this project uses into that project's `.env.local`. If a needed variable is missing or empty, stop. No `NEXT_PUBLIC_*`. Do not put a secret in a reply, a log, or a browser response. Do not commit `.secrets` or `.env.local`.

## 5. Call order

Send independent calls together. Show a block when that block is ready. Follow an order the requirements name. A failure explains only that block. Do not retry automatically.
