# MyDay Web App

> Layout and the zodiac 2×2 grid in this file are the English UI. This demo writes that UI in English. Providers and maps are in `specs/tech-spec.md`.

- A daily planning web app. It helps users start their day with personalized recommendations.
- This app does not serve mainland China. Maps use Google Maps only. Do not configure AMap. The request contracts are in `specs/tech-spec.md`.
- A browser location and a typed place are both checked first. When the place is in mainland China, show "Google Maps does not support this place", stay on page 3, and do not start fortune or weather. When the birth city is in mainland China, show the same message and start neither request.

## Overview
- Page 1, "Profile list": show the existing profile cards
- Page 2, "Create a new profile": the user enters personal details and saves a profile. After a successful save, return to page 1. The new profile appears in the list
- Page 3, "Plan my day": the user enters today's mood and activity, and the app gets the location (automatic, or a typed city when the location fails). After the location is usable, the user clicks "Plan my day" to get "today's fortune" and "today's weather"
- Page 4, "My day plan": show the fortune and weather from page 3. Generate today's outfit as text and as a portrait. After the user enters a cuisine, list nearby restaurants for the page 3 location

## Requirement
### Profile List
- A **card grid** to display all existing profiles (interaction cards)
- Each profile displays as a card, with name and photo
- Click a profile card, load that profile, and enter page 3, "Plan my day"
- Click "Create a new profile" to enter page 2

### Create a new profile
- User enters personal details
  - Name (required)
  - Birth city (required). One city, for example "Singapore"
  - Date of birth (required)
  - Upload a photo of yourself (required)
- Clicks "Save". If name, birth city, date of birth, or photo is missing, do not save. Stay on page 2 and mark the missing item. After a successful save, return to page 1. The new profile appears in the list

### Plan my day

This screen is page 3. The user confirms the profile and enters the current status here. This page only fetches "today's fortune" and "today's weather". Do not show them here. Show them on page 4, "My day plan". Enter page 4 only after both fetches succeed.

- Render **My profile** and **What am I into?** as separate full-width **section cards** stacked top → bottom (same visual language as My day plan)
- "My profile" section: show the current profile's name and photo. Do not edit or delete on this page
- "What am I into?" section:
  - Get the current location (default: the browser). When the location fails, require a manual city or district
  - Turn a typed city into a usable location first. That step is not part of the wait for fortune and weather. If the city cannot be resolved, stay on this page and say it must be entered again. The click starts neither request
  - While the location is still unusable, do not start "today's fortune" or "today's weather", and do not enter page 4
  - User manually enters "My mood now", e.g., "excited"
  - User manually enters "What I'm doing today", e.g., "travelling"
- After the location is usable, the user can click "Plan my day". While the location is unusable, the button cannot be clicked. The click starts "today's fortune" and "today's weather" together. After both have started, wait for them together. Do not start the second after the first finishes.
  - Fortune: using the zodiac, "My mood now", and "What I'm doing today", write "today's fortune", including things to do, things to avoid, and other playful tips. The chart location is the profile birth city, not the current location. If the birth city cannot be used, stay on this page, say the birth city must be changed, and start neither request. That step is not part of the wait. Write the fortune text with chat
  - Weather: get "today's weather" for the current location. The current location is only for weather and restaurants
- While waiting, this page shows one line: "Preparing insights for your day"
- If one of the two requests fails, stay on this page and say why. Do not enter page 4
- When both succeed, enter page 4, "My day plan"

### My day plan

This screen is page 4. The weather card and the zodiac card show the data page 3 already fetched. They do not fetch those two again, and they do not show "Preparing insights for your day". Outfit text and the outfit image start together. A failure in one block is explained on that block only.
Render as a **vertical stack of section cards** (one content kind per card). Do not merge section content into one flat column, and do **not** use a multi-column dashboard grid that mixes weather / zodiac / outfit / food side by side.

Each section card has: distinctive accent/surface, own heading, body, and its own loading / empty / error state.

- **Header** section card
  - "Hello, {name}" + photo if any
  - Display age, and the zodiac sign calculated from the date of birth
- **Weather** section card: show "today's weather" from page 3. Show the weather wording in the display language.
- **Zodiac insights** section card
  - Show only "today's fortune" from page 3. It already includes things to do, things to avoid, and playful tips. Do not generate it again on this page.
  - Render insights as an **inner 2×2 card grid** (4 cards) inside this section — not a single prose block:
    1. **Overall** — short vibe / chart summary for today
    2. **Things to do** — actionable suggestions
    3. **Things to avoid** — gentle cautions
    4. **Playful tips** — one or more fun tips
  - Each of the four cards has its own heading and body; stack to one column on narrow mobile if needed, 2 columns from `sm` up
  - The four cards show the fortune from page 3.
- **Outfit recommendation** section card. Use "today's weather", "today's fortune", "My mood now", and "What I'm doing today" from page 3. Start the text and the image together. Neither waits for the other.
  - Outfit text: write the recommendation and display it as text
  - Outfit image: a portrait in today's outfit. Use the profile photo as the person. If the photo cannot be read, show an error. Never generate the portrait without that photo.
  - Ensure the image is properly displayed with the visibility of the whole body (portrait aspect)
- **What to eat today** section card
  - The user enters a cuisine as text. Do not search before the user enters one
  - Use the location already obtained on page 3. Page 4 does not request the location again.
  - Search with the `searchText` example in the tech spec. `textQuery` is the text the user entered. The range is 3 km. Do not increase the radius. When a place photo exists, show it. If there is no photo, do not show a broken image.
  - Display results as a **true vertical list** (not a card grid): one row per restaurant, sorted by distance, scannable on mobile and desktop. An empty result says the user can change cuisine, or return to page 3 and pick the location again. Page 4 does not change the location. After returning to page 3, a new click uses the rules above for weather and fortune. The previous page 4 results are not reused. Never a blank list
  - **Each list row must include:** restaurant name, distance, and address. Do not show rating, review count, or a review snippet
  - On restaurant row click, open a floating popup with restaurant details and that region's map for location and navigation. Close on the close control, Escape, or backdrop click

### Non-functional requirements
- Capabilities this project uses: chat, image, weather, zodiac, and Google Maps. Do not configure AMap. Do not use Pokémon or directions.
- Install for this demo: Next.js, React, TypeScript, Tailwind, and Vitest for the tests. Store data under `data/`. Do not install React Query, Zustand, RHF, or Playwright.

## UI guidelines
Tone: **clean, neat, fun** — a light morning desk for planning the day, not a dense dashboard or marketing landing page.

### Visual direction
- **Palette (CSS variables):** Paper `#F7F9FC` · Mist `#E8F1F8` · Ink `#1B2430` · Soft slate `#5B6B7C` · Sun `#F4A261` (primary CTA / progress accent) · Fresh `#2BB673` (success only)
- **Surfaces:** soft sky-to-paper wash or subtle grain; avoid flat single-color pages and heavy purple / neon / dark-mode defaults
- **Type:** distinctive display for greetings / section titles (e.g. Outfit or similar geometric friendly face); clear sans for body and forms — never Inter, Roboto, Arial, or system UI stacks as the brand voice
- **Layout — section-card stack (required look & feel):**
  - Content screens use a **single-column vertical stack** of full-width **section cards**
  - Each card is visually distinct (border + light tint / accent per kind: e.g. sky for weather, star for zodiac, sun for outfit, leaf for food) so the page reads as a structured card layout, not one undifferentiated text wall
  - First screen (profile list): calm composition — brand + short prompt + profile **card grid** + primary create action
  - Do **not** arrange day-plan kinds in a multi-column masonry/dashboard grid
- **Interaction cards:** profile pick only — name + photo grid
- **Restaurant list (food section):** vertical list rows with name, distance, and address. A place photo may lead the row. Do not show rating or review text. Do not render restaurants as a multi-column card grid
- **Motion:** 2–3 intentional moments (section fade-in as content arrives, button press, dialog open/close); respect `prefers-reduced-motion`
- **Signature:** the progressive “day assembling” reveal — each stacked section card loads and appears on its own with light progress tips, not a single loading spinner for the whole page

### Page sections
Every screen stacks discrete section cards. Each has one purpose, one title, and its own loading / empty / error state.

| Screen | Sections (top → bottom) |
| --- | --- |
| Profile list | Header · Profile card grid · Create profile action |
| Create a profile | Header · Profile form section card · Save action |
| Plan my day | Header · My profile section card · What am I into? section card · Plan my day action |
| My day plan | Header · Weather · Zodiac insights · Outfit recommendation · What to eat today — each a full-width stacked section card with its own tone |

### Interaction & copy
- Primary actions use plain verbs that match outcomes (e.g. Save → saved confirmation; Plan my day → day plan)
- Empty, loading, and error states explain what to do next. Never show the word “Timeout”. A failed section shows its own message.
- Write the UI in English. This demo does not add a second locale.
- Keyboard: visible focus, Escape / backdrop closes dialogs; status is not color-only
- Desktop and mobile: no clipped controls; restaurant list rows stay full-width and scannable (image + text), never a restaurant card grid
