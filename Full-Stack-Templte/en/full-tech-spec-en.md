# Tech stack & dependencies

Shared across projects. Copy secrets into `.env.local` only — never commit real key values or `.env.local`. Product behavior belongs in each project’s requirements doc, not here.

## Stack

| Category | Tech | Version |
| --- | --- | --- |
| Framework | Next.js (App Router) | 16.3.0 |
| UI / language | React · TypeScript | 19.2.8 · 7.0.2 |
| Styles | Tailwind CSS | 4.3.1 |
| State / forms | React Query · Zustand · RHF + Zod + `@hookform/resolvers` | 5.101.4 · 5.0.15 · 7.85.0 + 4.4.3 + 5.7.1 |
| Storage | Local JSON (`data/`) — no DB | — |
| AI | `openai` SDK → OpenAI (ChatGPT) | 6.39.0 |
| Test | Vitest · RTL · Playwright | 4.1.9 · 16.3.2 · 1.61.1 |

Rules: minimal dependencies; prefer platform and stdlib; **no database** (local JSON under `data/` only unless a project spec says otherwise). Pick only the third-party capabilities a product needs — do not wire every service below by default.

## Minimum dependencies policy

- Use as few dependencies as possible — prefer the platform, stdlib, and existing stack before adding packages.
- Keep the architecture clean: no optional libs “just in case.”


## npm install

```bash
npm config set registry https://registry.npmjs.org/
# or one-off:
npm install --registry=https://registry.npmjs.org/
```

## 3rd party capabilities and API

### OpenAI Chat capability: gpt-5.4

- API Configuration
  - Model-name=`gpt-5.4`
  - Host=`api.openai.com`
  - API-Key=`OPENAI_API_KEY`
  - Base_URL=`https://api.openai.com/v1`
- Technical highlights
  - Endpoint: OpenAI `/chat/completions` via `openai` SDK (`gpt-5.4`)
  - Param: `max_completion_tokens` (**not** `max_tokens`)
  - On failure: return error to caller for component error + retry — never silent empty success
  - Product behavior: see project requirements doc

### OpenAI Image capability: gpt-image-1.5

- API Configuration
  - Model-name=`gpt-image-1.5`
  - Host=`api.openai.com`
  - API-Key=`OPENAI_API_KEY`
  - Base_URL=`https://api.openai.com/v1`
- Technical highlights
  - Endpoint: OpenAI `/images` via `openai` SDK (`gpt-image-1.5`) — generation and/or edit per project
  - Params (when supported): e.g. `size: "1024x1536"`, `quality: "low"` — use only API-documented sizes; never invent dimensions
  - **Performance tuning (≤20s component target; soft UI tip ~10s — see Performance):**
    - Keep **`quality: "low"`** and documented portrait size only — do not escalate quality or dimensions for “better” output
    - Request **`n: 1`**; no parallel duplicate image calls for the same logical output
    - **Downscale reference images on the server** before edit (e.g. cap long edge ~1024px, sensible JPEG/WebP)
    - **Short, structured prompts** — task summary only; avoid repeating long context already shown in UI
    - **Do not serialise** dependent text LLM and image on one critical path when text is already available — return text first, image immediately after when applicable
    - Server route: reuse a single `openai` client; outbound **HTTP timeout ~25s** — do **not** implement a separate **10s hard abort** while the UI still shows progress
    - On slow upstream: keep the request alive until timeout/error; surface “more time is needed” from the client after ~10s (per Performance) — never map a server cut-off to the word “Timeout”
    - Retry: **one** explicit user-triggered retry max per attempt path; do not auto-retry in a loop inside 20s
  - On failure: return error to caller — never silent empty image
  - Product behavior: see project requirements doc

### Google Maps Platform capability: Maps Platform

- API Configuration
  - Platform=`Google Maps Platform`
  - Host=`maps.googleapis.com`
  - API-Key=`GOOGLE_MAPS_API_KEY`
  - Project=`GOOGLE_MAPS_PROJECT`
  - Base_URL=per API (Places · Geocoding · Directions · Time Zone REST; Maps JavaScript · Embed via server-issued config)
- Technical highlights
  - APIs: Places nearby search, Geocoding (reverse / forward), Directions, Time Zone, Maps JavaScript, Embed
  - Do **not** add `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` — use a single `GOOGLE_MAPS_API_KEY` (server Embed/config only; place photos via server proxy when needed; no Maps key in client `img` URLs)
  - On failure: return error to caller — never silent empty success
  - Product behavior: see project requirements doc

### Open-Meteo capability: forecast

- API Configuration
  - Service=`Open-Meteo`
  - Host=`api.open-meteo.com`
  - API-Key=none (free, no key)
  - Base_URL=`https://api.open-meteo.com/v1` (env `OPEN_METEO_BASE_URL` optional)
  - Docs=`https://open-meteo.com/en/docs`
- Technical highlights
  - Endpoint: `GET /forecast` (e.g. `https://api.open-meteo.com/v1/forecast`)
  - Params (example): `latitude` · `longitude` · `current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m`
  - No auth header or query token required
  - On failure: return error to caller — never silent empty success
  - Product behavior: see project requirements doc

### Tripadvisor Terra capability: Terra API

- API Configuration
  - Service=`Tripadvisor Terra`
  - Host=`terra.tripadvisor.com`
  - API-Key=`TRIPADVISOR_API_KEY` (header `X-API-Key`)
  - Base_URL=`https://terra.tripadvisor.com/api` (env `TRIPADVISOR_BASE_URL`)
- Technical highlights
  - Endpoints: `GET /locations/nearby` · `GET /locations/{id}` · `GET /locations/{id}/reviews`
  - Params: `lat` + `lon` (not `latLong`); `radius`; `unit=KM` (uppercase)
  - Do **not** call Content API host `api.content.tripadvisor.com` with Terra key — Terra host only
  - Flatten localized name/review fields to plain strings before returning to UI
  - On failure: return error to caller or omit enrichment — never silent broken UI
  - Product behavior: see project requirements doc

### Resend capability: transactional email

- API Configuration
  - Service=`Resend` (or SMTP)
  - Host=Resend API or `SMTP_URL`
  - API-Key=`RESEND_API_KEY` (or `SMTP_URL` for SMTP)
  - Base_URL=per Resend / SMTP provider docs
- Technical highlights
  - Transactional email (e.g. password reset, notifications)
  - On failure: return error to caller — do not claim send success without provider ack
  - Product behavior: see project requirements doc

### Finnhub capability: U.S. stock market data

- API Configuration
  - Service=`Finnhub`
  - Host=`finnhub.io`
  - API-Key=`FINNHUB_API_KEY` (query param `token`)
  - Base_URL=`https://finnhub.io/api/v1`
- Technical highlights
  - Stock quotes, company news, market calendar, basic financials · free tier · [docs](https://finnhub.io/docs/api)
  - Example: `GET /quote?symbol=AAPL&token=…`
  - On failure: return error to caller — never silent empty success
  - Product behavior: see project requirements doc

### Pandorium MCP capability: Streamable HTTP MCP

- API Configuration
  - Service=`Pandorium MCP`
  - Host=`pandorium.app`
  - API-Key=none
  - Base_URL=`https://pandorium.app/mcp` (env `PANDORIUM_MCP_URL`)
- Technical highlights
  - Transport: Streamable HTTP (JSON-RPC POST, SSE `data:` responses)
  - Tools (examples): `tools/list`, `get_planet_positions`, `calculate_natal_chart`, `get_current_transits`
  - Natal / transit tools require `birthData` plus timezone — do not call with date of birth alone
  - On failure: return error to caller — never silent empty success
  - Docs: [pandorium-mcp](https://github.com/Naitellos/pandorium-mcp)
  - Product behavior: see project requirements doc

### TCGdex capability: Pokémon TCG catalog

- API Configuration
  - Service=`TCGdex`
  - Host=`api.tcgdex.net`
  - API-Key=none (free, no key)
  - Base_URL=`https://api.tcgdex.net/v2/{lang}` (env `TCGDEX_BASE_URL` optional, default `https://api.tcgdex.net/v2`)
  - Docs=`https://tcgdex.dev/` · REST=`https://tcgdex.dev/rest`
- Technical highlights
  - HTTPS GET only · JSON responses · language path segment (e.g. `en`, `fr`, `ja`)
  - Examples: `GET …/en/cards/{id}` · `GET …/en/cards?name=pikachu` · sets / series / illustrators
  - Catalog, sets, images, market prices (TCGPlayer USD, Cardmarket EUR), short price trends
  - Optional SDK: `@tcgdex/sdk` — open-source · [docs](https://tcgdex.dev/)
  - On failure: return error to caller — never silent empty success
  - Product behavior: see project requirements doc

## Environment

Copy keys into `.env.local` (gitignored; do not commit). Do not hardcode in source. Spec lists **env codes only** — fill values locally.

```env
# ChatGPT OpenAI
OPENAI_API_KEY=
OPENAI_BASE_URL=
OPENAI_CHAT_MODEL=
OPENAI_IMAGE_MODEL=
OPENAI_KEY_MGMT_SITE=

# Maps
GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_HOST=
GOOGLE_MAPS_PROJECT=
GOOGLE_MAPS_BASE_URL=
GOOGLE_MAP_KEY_MGMT_SITE=

# Tripadvisor Places
TRIPADVISOR_API_KEY=
TRIPADVISOR_HOST=
TRIPADVISOR_BASE_URL=
TRIPADVISOR_KEY_MGMT_SITE=

# Email / Resend
RESEND_API_KEY=
RESEND_HOST=
RESEND_BASE_URL=
RESEND_KEY_MGMT_SITE=
SMTP_URL=

# U.S. Stock
FINNHUB_API_KEY=
FINNHUB_HOST=
FINNHUB_BASE_URL=
FINNHUB_DOCS=

# Weather (no key)
OPEN_METEO_HOST=
OPEN_METEO_BASE_URL=
OPEN_METEO_DOCS=

# Pokémon TCG catalog (no key)
TCGDEX_BASE_URL=
TCGDEX_DOCS=

# MCP / optional
PANDORIUM_MCP_URL=

# No key: Open-Meteo, Pandorium (default URL), TCGdex
```

## Performance

Image generation has additional tuning — see the Performance tuning section under the OpenAI Image capability.

**Latency (all components — one contract only):**

- Soft tip: after ~**10s**, show “more time is needed” while work continues (never the word “Timeout”)
- Target: each component ready within **≤20s**
- Do **not** implement a separate conflicting 10s hard timeout that abandons the component early without a user-visible state

**Loading strategy (all components):**

- Fetch in **parallel** across independent components
- **Stream / render** each piece as it completes — do not block the whole view on the slowest component
- Show **per-component** tips and progress while work is pending

**Fallbacks:**

- Use **local fallbacks** only when upstream fails or the component is still empty after the user-visible wait path
- Do **not** silently swap to a fallback at ~10s while the UI still implies live generation

## Implementation pitfalls (must follow)

Cross-check when implementing against the project requirements doc:

| Area | Contract |
| --- | --- |
| **Chat** | Use `max_completion_tokens` — **not** `max_tokens` on OpenAI chat surfaces |
| **Image (OpenAI)** | Documented sizes and `quality: low` only when applicable; never silent image failure; see Performance tuning under OpenAI Image |
| **Maps** | Single server `GOOGLE_MAPS_API_KEY` — no `NEXT_PUBLIC_*` |
| **Tripadvisor** | Terra host only; flatten localized strings before UI |
| **Latency** | Soft tip ~10s · target ready ≤20s — **one contract only** (see Performance — all components) |
| **Secrets** | Env vars only in `.env.local` — never commit real keys in specs or repo |

## Localization note (i18n)

This stack is **not bound** by the i18n default requirement from the always-on rules. User-facing copy may be hard-coded; the localization strategy for a project (supported locales, message catalogs, etc.) is decided by that project's requirements doc.

## Quality bar & testing

Minimum bar every project on this stack must meet (based on the `common-test-strategy` baseline, not its full default checklist):

- Unit tests for business logic; **100%** coverage of the changed critical path
- Integration tests across API / data-boundary paths
- At least one critical user journey covered by E2E in a real browser
- Tests use a temporary data directory — never production data
- No database: persistence is local JSON under `data/`

**Exemption:** this stack is not bound by the full default checklist of `common-test-strategy`; stricter project-specific test strategies may be defined in each project's requirements / test-strategy doc.
