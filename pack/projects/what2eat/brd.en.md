# What2Eat — Business Requirements

## What it is

What2Eat is an AI-assisted meal decision helper for people who cannot decide what to eat. It helps users pick a dish or nearby place from preferences, constraints, location, weather, and time of day. It combines LLM (Quanzil or OpenAI per region), agents, MCP services, and maps/places data (Google Maps or AMap per `pack/specs/tech-spec.*.md` section 1). It does not place orders, process payments, or replace restaurant menus, allergen labels, or medical dietary advice.

## Who it’s for

Primarily individuals and small groups who want a fast, constrained recommendation (“what should we eat tonight?”). Office lunch groups, travelers in a new city, and households with mixed diets may also use it.

## How it fits together

- **Web app** — account, preference profile, “decide now” flow, saved places, and explanation views. The browser talks only to the app backend.
- **eat-places-mcp** — geocodes the user area and retrieves nearby restaurants / POIs, hours, and basic place details from approved map providers (Google Maps or AMap per the regional matrix in `pack/specs/tech-spec.*.md`).
- **what2eat-agent** — ranks options using MCP data plus RAG knowledge; separates place facts, assessments, and suggestions.
- **food-pref-rag** — cuisine patterns, common dietary constraints, and meal-timing heuristics for the agent.

## Features

### User Account

- Register a new user (name, email, optional photo, default city/area, password)
- Log in with email and password
- Reset password via email
- Header in the top-right corner of authenticated pages: “Hello, [Username]” + avatar + Logout

### Preference Profile

- Capture cuisines liked / disliked, spice tolerance, budget band, party size default, and hard constraints (e.g. vegetarian, no pork, nut allergy flag as user-stated only)
- Save meal contexts: weekday lunch, weekend dinner, quick bite, celebration
- Edit preferences anytime; show last-updated time on the profile

### Decide Now

- Start from current location (or pinned area) + meal context + optional “I’m craving…” text
- Apply time-of-day and open-now filters when hours are available
- Return a short ranked list (not an endless feed); allow reshuffle with the same constraints
- Block empty results with a clear reason (too narrow filters, no places nearby, provider failure)

### Option Cards

- Show recommended places/dishes in a responsive list or grid
- Each card shows name, cuisine tags, distance/ETA when available, price band, open status, match reasons, and clear placeholders when data is missing
- Click a card to open Place Details; click a map pin to open Map Focus

### Place Details

- Pop-up or screen with address, hours, contact when available, and match explanation; close via Close, Escape, or outside click
- Link to Why This Pick analysis; show sources and last update
- Save to Favorites or Dismiss with confirmation

### Match Classification

- Fit score: Green (strong match), Yellow (partial / trade-offs), Red (weak / conflicts with hard constraints) — with text label and description
- Place readiness: Green (likely open / reachable), Yellow (uncertain hours or distance), Red (closed / unreachable / data missing) — with text label and description
- Color alone is never enough

### Places MCP — eat-places-mcp

- Built-in MCP that resolves location and retrieves nearby dining options and place details
- Normalizes provider data; flags stale, missing, or conflicting hours
- Returns standardized place identity, distance, open status when known, sources, last update, and confidence

### Decision Alerts

- Show warnings on Option Cards and Place Details
- Cover closed now, long travel time, incomplete hours, and allergen uncertainty
- Clearly highlight when a top pick becomes weaker after refresh

### Favorites & History

- Save liked places; hide repeatedly dismissed ones for a cooling-off period
- Show recent decisions with what was chosen and whether the user marked “went / skipped”
- Allow re-run of a past decision with updated live place status

### Decision Analytics Agent — what2eat-agent

- Analyzes candidates using eat-places-mcp and food-pref-rag
- Considers preferences, constraints, distance, open status, weather (if available), party size, and budget
- Distinguishes confirmed place facts, assessments, and suggestions; never presents guesses as menu or allergen facts

### Why This Pick

- Screen for the selected option: match reasons, trade-offs, alternatives, sources, last update, and confidence
- Chat about the selected place or “what else nearby?” (selection stays in conversation context)
- Agent prompt stored in a separate, version-controlled file

### Group Mode

- Collect short preference votes from multiple participants (same session)
- Merge hard constraints (union of excludes); rank by shared likes
- Show disagreement notes when the group cannot converge

### Food Preference RAG — food-pref-rag

- Knowledge base of cuisine pairings, meal-timing heuristics, and common dietary constraint patterns
- Load generic knowledge at app launch; load locale/cuisine notes on demand for a new city or cuisine family
- Prefer well-sourced references; each item includes source and dates when available
- Never treat RAG content as medical or legal dietary advice

### Data Quality

- Always show when place information was last updated
- Flag stale, incomplete, unavailable, or conflicting data; distinguish user preference, place fact, and model suggestion
- AI recommendations must not be presented as official restaurant commitments or health guidance
- One provider failure must not take down the whole app

### Performance

- Show last-known favorites immediately while nearby search loads
- Meet latency targets for decide-now, search, details, and AI responses
- Cache with last-updated time; never label cached open/closed status as guaranteed real-time
- Use timeouts, retries, and fallbacks; avoid unnecessary repeated API/MCP/RAG/LLM calls

## Success looks like

A user can set preferences once, get a small set of suitable options quickly, understand why each pick fits (and what is uncertain), ask follow-up questions in natural language, and leave ready to go—while still checking the restaurant’s own menu, hours, and allergen information before ordering.
