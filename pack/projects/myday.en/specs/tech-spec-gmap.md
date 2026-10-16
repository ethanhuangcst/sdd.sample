# MyDay tech spec


## 1. Principles
- Work in the workspace root. Do not reference other projects. Do not write code or docs outside this workspace.
- The workspace root holds `specs/` and the app directory `web/`. Put Next.js code only in `web/`.
- No persistent storage for this demo. Do not write `data/`.
- Install only what this project needs. Do not install React Query, Zustand, RHF, or Playwright.

## 2. Stack

| Role | Technology |
| --- | --- |
| App | Next.js 16, React 19, TypeScript, Tailwind CSS 4 |
| Chat | `openai` SDK, Quanzil chat completions |
| Image | Gemini `generateContent`, not OpenAI images API |
| Tests | Vitest |


## 3. Capabilities

### Chat

```env
QUANZIL_API_KEY=
QUANZIL_BASE_URL=https://quanzil.com/v1
QUANZIL_CHAT_MODEL=gpt-6-astra
```

Use the `openai` SDK with the key and base URL above. Do not set `OPENAI_API_KEY` or `OPENAI_BASE_URL`. Use `max_completion_tokens`, not `max_tokens`. An empty `choices[0].message.content` counts as failure.

### Image

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

`POST` that URL. Header `Authorization: Bearer <QUANZIL_API_KEY>`. Body includes `generationConfig.responseModalities: ["IMAGE"]`. With a photo, `parts` include text and `inlineData`. Use the photo MIME type. Scale the long edge to about 1024px. A 1×1 image returns 400. Do not set `imageConfig.imageSize`. The result is in `candidates[0].content.parts[].inlineData`. Missing `inlineData` counts as failure. Do not drop the photo from the request.

### Google Maps

```env
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_GEOCODE_URL=https://maps.googleapis.com/maps/api/geocode/json
GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL=https://places.googleapis.com/v1/places:searchText
```

- Geocode: `GET ${GOOGLE_MAPS_GEOCODE_URL}?address={address}&key=`. Success when `status` is `OK`. WGS-84, order lat,lng.
- Restaurants: `POST` searchText. Headers `Content-Type: application/json`, `X-Goog-Api-Key`, `X-Goog-FieldMask: places.displayName,places.formattedAddress,places.location`
- Body: `textQuery` plus `locationBias.circle`. `circle` is required. Radius 5000 m. One call only.
- Compute distance from returned coordinates. No per-row detail calls.
- Browser link: `https://www.google.com/maps/dir/?api=1&destination=lat,lng`. No JS API. Keep the key server-side only. No `NEXT_PUBLIC_*`.

### Weather

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

- `GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,weather_code`. WGS-84. Reuse coordinates; do not geocode again for weather.
- Map `weather_code` with the table below before display. Unknown codes show "Unknown".

| Code | Label |
| --- | --- |
| 0 | Clear |
| 1 | Mainly clear |
| 2 | Partly cloudy |
| 3 | Overcast |
| 45 | Fog |
| 48 | Depositing rime fog |
| 51 | Light drizzle |
| 53 | Drizzle |
| 55 | Dense drizzle |
| 56 | Light freezing drizzle |
| 57 | Dense freezing drizzle |
| 61 | Slight rain |
| 63 | Moderate rain |
| 65 | Heavy rain |
| 66 | Light freezing rain |
| 67 | Heavy freezing rain |
| 71 | Slight snow |
| 73 | Moderate snow |
| 75 | Heavy snow |
| 77 | Snow grains |
| 80 | Slight rain showers |
| 81 | Rain showers |
| 82 | Violent rain showers |
| 85 | Slight snow showers |
| 86 | Heavy snow showers |
| 95 | Thunderstorm |
| 96 | Thunderstorm with slight hail |
| 99 | Thunderstorm with heavy hail |

## 4. Install

### Step 1: Dependencies

- If `web/package.json` is missing, from the workspace root run (empty or missing `web/` is OK). Do not pause for scaffold prompts. Mainland China, Hong Kong, Macau:
  ```
  npm_config_registry=https://registry.npmmirror.com npx create-next-app@16 web --yes --ts --tailwind --eslint --app --no-src-dir --use-npm
  ```
  Other regions: same command without `npm_config_registry`.
  Do not run an unnecessary `npm install` after this.
- Then in `web/`, add packages. In CN/HK/MO add `--registry=https://registry.npmmirror.com`:
  ```
  npm install openai
  npm install -D vitest jsdom @testing-library/react @testing-library/jest-dom
  ```

- If `web/package.json` and lockfile exist, run `npm ci` in `web/` instead of `create-next-app`. Add missing packages with the commands above.
- On peer dependency failure, retry that `npm install` with `--legacy-peer-deps`. Do not change `@types/node`.

### Step 2: Secrets

- Secrets live in `specs/.secrets`, filled in by the operator.
- Copy only variable names from section 3 code blocks into `web/.env.local`. Skip extra keys in `.secrets`. Stop if any required value is missing. No `NEXT_PUBLIC_*`. Never log or expose secrets.

## 5. Call order

Start independent calls together. Show each block when it finishes. The outfit image waits for the outfit text. No auto-retry.
