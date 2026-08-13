# PokeTrade — Business Requirements

## What it is

PokeTrade is an AI-assisted Pokémon Trading Card Game (TCG) collection and trade helper. It helps collectors look up cards, track owned and wanted lists, compare market prices, and judge whether a trade or purchase is reasonable. It combines LLMs, agents, MCP services, and catalog/price data (e.g. TCGdex). It does not execute trades, process payments, or replace official Pokémon Company rules, store policies, or marketplace terms.

## Who it’s for

Primarily Pokémon TCG collectors and casual traders who want clearer card identity and price context before buying, selling, or swapping. Parents helping kids manage collections and local shop regulars may also use it.

## How it fits together

- **Web app** — account, collection, want list, card cards, trade scratchpad, and analysis views. The browser talks only to the app backend.
- **poke-catalog-mcp** — searches cards and retrieves set, rarity, images, and market price snapshots from approved catalog providers (primary: TCGdex).
- **poke-trade-agent** — analyzes value and trade fairness using MCP data plus RAG knowledge; separates catalog facts, assessments, and price predictions.
- **poke-rules-rag** — set releases, legality notes, grading basics, and common trade pitfalls for the agent.

## Features

### User Account

- Register a new user (name, email, optional photo, collector experience level, password)
- Log in with email and password
- Reset password via email
- Header in the top-right corner of authenticated pages: “Hello, [Username]” + avatar + Logout

### Collection Cards

- Show owned cards in a responsive grid, filterable by set, rarity, and language
- Each card shows name, set, number, condition (user-entered), quantity, latest price snapshot, thumbnail (user photo or catalog image), and clear placeholders when data is missing
- Click a card to open Card Details; click a set to open Set Overview

### Want List

- Add cards the user is looking for; show quantity wanted and max willing price (optional)
- Highlight when a want-list card appears in another user’s public trade offer (if sharing is enabled later)
- Block duplicate want entries for the same card identity unless condition variants differ

### Add / Search Cards

- Search by card name, set code, or card number (via poke-catalog-mcp)
- Choose the correct printing when multiple matches appear (language, set, variant)
- Add to Collection or Want List; record condition and quantity
- Manually upload a card picture (camera or file) when adding or editing an owned card; store the photo with that collection entry
- Prefer the user photo as the card thumbnail when present; fall back to the catalog image; show a placeholder if neither is available
- Validate file type and size; allow replace or remove of the uploaded picture with confirmation

### Card Details

- Pop-up or screen with full catalog details, images (user photo and/or catalog art), and price history summary; close via Close, Escape, or outside click
- Upload, replace, or remove the card picture from details
- Link to Trade Analysis; show price sources and last update
- Remove from Collection / Want List with confirmation

### Price Status Classification

- Price signal: Green (stable / near recent average), Yellow (volatile or incomplete data), Red (outlier / unreliable / conflicting sources) — with text label and description
- Trade fairness: Green (balanced), Yellow (skewed), Red (strongly one-sided) — with text label and description
- Color alone is never enough

### Catalog / Price MCP — poke-catalog-mcp

- Built-in MCP that searches cards and retrieves catalog metadata and market prices (TCGPlayer USD, Cardmarket EUR where available)
- Normalizes provider data; flags stale, missing, or conflicting prices
- Returns standardized card identity, price snapshots, sources, last update, and confidence

### Collection Alerts

- Show price moves and data-quality warnings on Collection Cards and Card Details
- Cover large price swings, missing price, set reprints, and condition-sensitive caveats
- Clearly highlight when a tracked card’s price signal becomes more severe

### Trade Scratchpad

- Select cards offered and cards requested; show side totals and gap
- Support multi-card packages; show currency and source per line
- Save drafts; clear with confirmation

### Trade Analytics Agent — poke-trade-agent

- Analyzes proposed trades using poke-catalog-mcp and poke-rules-rag
- Considers print rarity, condition assumptions, language, recent averages (avg1/7/30 when available), and data gaps
- Distinguishes catalog facts, assessments, and predictions; never presents market estimates as guaranteed sale prices

### Trade Analysis

- Screen for the selected trade draft: side values, gap, volatility notes, collector impact, sources, last update, and confidence
- Chat about the selected trade (trade draft stays in conversation context)
- Agent prompt stored in a separate, version-controlled file

### Set Overview

- Screen for the selected set: identity, card count, owned vs missing, price coverage, sources, last update, and confidence
- Chat about the selected set (set stays in conversation context)
- Agent prompt stored in a separate, version-controlled file

### Rules / Market RAG — poke-rules-rag

- Knowledge base of common TCG trade practices, grading basics, reprint awareness, and marketplace caveats
- Load generic knowledge at app launch; load set-specific notes on demand when a new set is accessed
- Prefer authoritative or well-sourced references; each item includes source and dates when available

### Data Quality

- Always show when price and catalog information was last updated
- Flag stale, incomplete, unavailable, or conflicting data; distinguish list price, last sale, and average
- AI value judgments must not be presented as official Pokémon Company, shop, or marketplace valuations
- One provider failure must not take down the whole app

### Performance

- Show stored collection info immediately while price updates load
- Meet latency targets for dashboard, search, details, set overview, and AI responses
- Cache with last-updated time; never label cached prices as live market
- Use timeouts, retries, and fallbacks; avoid unnecessary repeated API/MCP/RAG/LLM calls

## Success looks like

A collector can manage collection and want lists in one place, quickly see which cards or trades need attention, understand the main price reasons with sources and confidence, ask follow-up questions in natural language, and leave better prepared for a fair swap—while still treating shops, marketplaces, and personal inspection as the final authority.
