# Where2Play — Business Requirements

## What it is

Where2Play is an AI-assisted activity and outing finder. It helps people decide where to play—parks, indoor play spaces, sports courts, family attractions, and casual hangouts—based on location, weather, time available, age group, and mood. It combines LLM (Quanzil or OpenAI per region), agents, MCP services, and maps/places data (Google Maps or AMap per `pack/specs/tech-spec.*.md` section 1). It does not book tickets, process payments, or replace venue rules, safety notices, or official hours.

## Who it’s for

Primarily families, friends, and small groups looking for “something to do nearby” with clear constraints (kids’ ages, rain, short window). Solo users planning a walk or light activity may also use it.

## How it fits together

- **Web app** — account, play profile, “find something” flow, saved places, outing plans, and explanation views. The browser talks only to the app backend.
- **play-places-mcp** — geocodes the user area and retrieves nearby venues / POIs, hours, and basic place details from approved map providers (Google Maps or AMap per the regional matrix in `pack/specs/tech-spec.*.md`); may attach local weather when relevant.
- **where2play-agent** — ranks outing options using MCP data plus RAG knowledge; separates place facts, assessments, and suggestions.
- **play-context-rag** — activity-type heuristics, weather suitability, and age-band notes for the agent.

## Features

### User Account

- Register a new user (name, email, optional photo, default city/area, password)
- Log in with email and password
- Reset password via email
- Header in the top-right corner of authenticated pages: “Hello, [Username]” + avatar + Logout

### Play Profile

- Capture preferred activity types (park, playground, museum, sports, indoor soft play, scenic walk), energy level, budget band, and mobility notes (stroller-friendly, step-free when user-stated)
- Save age bands for household members (ranges only; no medical data required)
- Edit profile anytime; show last-updated time on the profile

### Find Something

- Start from current location (or pinned area) + time budget + weather-aware toggle + optional mood text
- Apply open-now / still-open-for-N-minutes filters when hours are available
- Return a short ranked list of places or outing ideas; allow reshuffle with the same constraints
- Block empty results with a clear reason (too narrow filters, bad weather for outdoor-only, provider failure)

### Place Cards

- Show recommended places in a responsive grid or list, sorted by fit then distance
- Each card shows name, activity tags, distance/ETA when available, indoor/outdoor, weather fit, open status, and clear placeholders when data is missing
- Click a card to open Place Details; click an area name to open Area Snapshot

### Place Details

- Pop-up or screen with address, hours, suitable age notes (heuristic), and match explanation; close via Close, Escape, or outside click
- Link to Why This Outing analysis; show sources and last update
- Save to Favorites or Dismiss with confirmation

### Fit Classification

- Outing fit: Green (strong match), Yellow (partial / trade-offs), Red (weak / conflicts with hard constraints) — with text label and description
- Conditions: Green (weather and hours look fine), Yellow (marginal weather or uncertain hours), Red (closed / severe weather / unreachable) — with text label and description
- Color alone is never enough

### Places / Conditions MCP — play-places-mcp

- Built-in MCP that resolves location and retrieves nearby play-relevant places and, when useful, local weather summary
- Normalizes provider data; flags stale, missing, or conflicting hours
- Returns standardized place identity, distance, indoor/outdoor when known, sources, last update, and confidence

### Outing Alerts

- Show warnings on Place Cards and Place Details
- Cover closing soon, rain/heat advisories for outdoor picks, incomplete hours, and long travel vs time budget
- Clearly highlight when a top pick becomes weaker after refresh

### Favorites & Outing Plans

- Save liked places; group into a simple outing plan (ordered stops + optional buffer time)
- Show recent outings with went / skipped feedback
- Allow re-run of a past search with updated weather and open status

### Outing Analytics Agent — where2play-agent

- Analyzes candidates using play-places-mcp and play-context-rag
- Considers activity preference, age bands, weather, distance, open status, indoor/outdoor, and time budget
- Distinguishes confirmed place facts, assessments, and suggestions; never presents guesses as safety certifications or official age limits

### Why This Outing

- Screen for the selected option: match reasons, weather notes, trade-offs, nearby alternatives, sources, last update, and confidence
- Chat about the selected place or “what if it rains?” (selection stays in conversation context)
- Agent prompt stored in a separate, version-controlled file

### Area Snapshot

- Screen for the selected area or cluster: density of play options, weather summary, notable constraints, sources, last update, and confidence
- Chat about the selected area (area stays in conversation context)
- Agent prompt stored in a separate, version-controlled file

### Group Mode

- Collect short votes from multiple participants (same session): activity type, energy, indoor/outdoor preference
- Merge hard constraints; rank by shared fit
- Show disagreement notes when the group cannot converge

### Play Context RAG — play-context-rag

- Knowledge base of activity-type heuristics, weather suitability patterns, and age-band outing notes
- Load generic knowledge at app launch; load locale notes on demand for a new city or venue type
- Prefer well-sourced references; each item includes source and dates when available
- Never treat RAG content as medical, childcare, or legal safety advice

### Data Quality

- Always show when place and weather information was last updated
- Flag stale, incomplete, unavailable, or conflicting data; distinguish user preference, place fact, and model suggestion
- AI recommendations must not be presented as official venue commitments or safety guarantees
- One provider failure must not take down the whole app

### Performance

- Show last-known favorites immediately while nearby search and weather load
- Meet latency targets for find-something, details, area snapshot, and AI responses
- Cache with last-updated time; never label cached open/closed or weather as guaranteed real-time
- Use timeouts, retries, and fallbacks; avoid unnecessary repeated API/MCP/RAG/LLM calls

## Success looks like

A user can set play preferences once, get a small set of suitable nearby outings quickly, understand weather and time trade-offs with sources and confidence, ask follow-up questions in natural language, and leave ready to go—while still checking the venue’s own hours, rules, and safety notices before arriving.
