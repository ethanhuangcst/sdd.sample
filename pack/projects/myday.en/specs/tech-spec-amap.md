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

`gpt-6-astra` counts reasoning tokens inside `max_completion_tokens` before it writes `choices[0].message.content`. The word and character limits in the requirement prompts do not set this cap. Set `max_completion_tokens` to 1200 for the status summary, the itinerary, and the outfit text. An empty `choices[0].message.content` is failure, including when `finish_reason` is `length` because reasoning used the whole cap.

### Image

```env
QUANZIL_IMAGE_MODEL=gemini-3.1-flash-lite-image
QUANZIL_IMAGE_URL=https://quanzil.com/v1beta/models/gemini-3.1-flash-lite-image:generateContent
```

`POST` that URL. Header `Authorization: Bearer <QUANZIL_API_KEY>`. Body includes `generationConfig.responseModalities: ["IMAGE"]`. With a photo, `parts` include text and `inlineData`. Use the photo MIME type. The page-1 photo is small: the user controls the upload, long edge within about 2048px and file within 2MB. The app does not scale images. Wrap the image POST in a 75-second timeout; a timeout fails that block, no retry. A 1×1 image returns 400. Do not set `imageConfig.imageSize`. The result is in `candidates[0].content.parts[].inlineData`. Missing `inlineData` counts as failure. Do not drop the photo from the request.

### AMap

```env
AMAP_API_KEY=
AMAP_HOST=https://restapi.amap.com
```

- Geocode: `GET {AMAP_HOST}/v3/geocode/geo?address={address}&key=`
- Restaurants: `GET {AMAP_HOST}/v5/place/around?location={lng},{lat}&keywords={cuisine}&types=050000&radius=5000&key=`
- Success: HTTP 200, `status` `1`, `infocode` `10000`. Coordinates are GCJ-02, order lng,lat.
- Restaurant search uses GCJ-02. Convert to WGS-84 before Open-Meteo, using local iteration; do not add a coordinate library.
- One around search only. Use name, distance, address, type, and phone from that response. No per-row detail calls.
- Browser link: `https://uri.amap.com/marker?position=lng,lat`. No JS API. Keep the key server-side only.

### Time zone

After geocoding succeeds, map the returned `adcode` prefix to an IANA time zone id. Do not call a time zone API. Do not install `geo-tz`, `tz-lookup`, or similar packages.

| `adcode` prefix | IANA |
| --- | --- |
| 65 | `Asia/Urumqi` |
| 81 | `Asia/Hong_Kong` |
| 82 | `Asia/Macau` |
| 71 | `Asia/Taipei` |
| Any other successful result | `Asia/Shanghai` |

- Use `Intl.DateTimeFormat` with `timeZone` set to that IANA id for local date and time at the place. Do not use the browser system time zone instead. Do not estimate from longitude divided by 15.
- For age, "today" uses the local calendar date in that IANA id. Subtract one year if this year's birthday has not occurred yet.
- Pass this IANA id, local time, and coordinates when entering page 2.

### Weather

```env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
```

- `GET {OPEN_METEO_BASE_URL}/forecast?latitude={lat}&longitude={lng}&current=temperature_2m,weather_code&timezone={IANA}`. WGS-84. Use the same IANA id from the time zone section. Reuse coordinates; do not geocode again for weather.
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

### Zodiac

Do not call a horoscope service and do not install a zodiac package. Use Gregorian month and day only, not birth time or birth year. Boundaries match `specs/req-spec.md` and this list: Capricorn 12/22–1/19, Aquarius 1/20–2/18, Pisces 2/19–3/20, Aries 3/21–4/19, Taurus 4/20–5/20, Gemini 5/21–6/21, Cancer 6/22–7/22, Leo 7/23–8/22, Virgo 8/23–9/22, Libra 9/23–10/23, Scorpio 10/24–11/22, Sagittarius 11/23–12/21.

- Implement a local function in `web/`. Unit-test 12/21, 12/22, 1/19, 1/20, and the day before and after every other boundary.
- Show English zodiac names in the UI.

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
