# How2Cook — Requirements Spec (English)

- Based on `brd-en.md` in this folder. The BRD owns the scope; this file turns each item into verifiable behavior
- UI is English only. No i18n; no second language is planned
- Wiring details are in [`../../tech-spec-en.md`](../../tech-spec-en.md): Chinese recipes under "HowToCook — local stdio MCP", Western and other under "RecipeWiki — remote MCP" and "TheMealDB — REST API", translation under "Chat LLM"
- Capabilities this project uses: the chat LLM (translation only) and the three recipe sources. No image, map, weather, zodiac, or Pokémon

## Data sources and translation

- Cuisines come in three groups: Chinese (HowToCook), Western and other (RecipeWiki and TheMealDB). Every recipe is labeled with its source name
- Recipe info returned by the recipe MCPs and APIs is translated into the UI language by the "Chat LLM" before display. When the source language matches the UI language, no translation happens
- Translation scope: dish names, ingredient names, amount wording, steps, and category names. Source product names (HowToCook, RecipeWiki, TheMealDB) are not translated
- Translations are cached in `data/` keyed by source + recipe id + language, with a last-updated time; a cache hit does not call the LLM again
- Translation prompt:

```text
You are a translator. Translate the recipe content below into English.
Translate the source text only. Keep ingredient amounts and units. Do not add steps, ingredients, or advice the source does not have.
On first mention, give the dish an English name and keep the original name in parentheses.
```

- Fields a source does not have show placeholders (for example, "Not provided by the source"); nothing is invented
- A single source failing affects only that source's section; the page explains why, and other sources keep working

## User account

- Register: username (required, 2–32 characters), email (required, format checked), password (required, at least 8 characters with letters and digits). Failed validation stays on the page and marks the failed fields
- Log in: email and password. Errors use one generic hint; the page does not say whether the email or the password was wrong
- Forgot password: entering the registered email sends a reset email; registered and unregistered emails get the same submitted hint
- Header: logged in shows "Hello, {username}" and a Logout button; logged out shows Log in and Register entries
- Favorites, shopping list, and cooked history belong to the user and are stored on this machine in `data/`. Passwords are stored hashed, never in plain text

## Find a dish (home)

The home page, top to bottom: header, cuisine switch, category browse, search, find-by-ingredients, footer.

- Cuisine switch: Chinese / Western and other. Switching keeps the typed search text
- Category browse: recipe cards (image, name, source label, time or servings when the source has them) by category within the current cuisine
  - Chinese categories come from the HowToCook category enum (seafood, breakfast, meat, vegetable, staple, soup, dessert, drinks, seasonings, semi-finished)
  - Western and other: the RecipeWiki tag list merged with the TheMealDB category list, deduplicated
- Search: keyword search by dish name within the current cuisine; each source is queried once and results merge with deduplication. No results show "No recipes found; try other keywords"
- Find by ingredients: the user lists ingredients on hand; the words are used as search keywords within the current cuisine. Results rank by match strength; no results explain that other ingredients may work
- Surprise me: Chinese uses the HowToCook per-headcount recommendation and takes one dish; Western and other use TheMealDB random endpoint. The result opens "Recipe detail" directly
- Every card shows its source. Tapping opens "Recipe detail"

## Recipe detail

- Shows: image, name, cuisine and category, source, servings, prep/cook time (when the source has it)
- Ingredient list: name and amount
- Portion scaling: a headcount input (default: the source's servings). Changing it rescales ingredient amounts by ratio, computed locally with no network call. Quantities embedded in step text are not scaled; the page says so in one line
- Steps: numbered instructions
- Actions: "Add to shopping list", "Favorite", and "Mark as cooked" take effect immediately, show "Added", and the hint fades out after 3 seconds
- Data quality: shows source and last-updated time; translated text shows "Content translated by AI; see the original for the authoritative text"

## Shopping list

- The list groups items by recipe, each group listing that recipe's ingredients and amounts
- Tapping an item strikes it through; removing a recipe removes its group
- Ingredients with the same name across recipes are not merged

## Favorites and history

- "My favorites" shows favorited recipes in a grid; tapping opens the detail
- Cooked history shows recently cooked dishes with dates; each entry reopens the detail

## Data quality

- Every dish shows its source and last-updated time
- Translated text carries the translation label; missing fields show placeholders
- A single source failing affects only that source's section; the page explains why

## Performance

- Cached translations display immediately while categories and searches load
- Latency targets for find-a-dish and detail are met
- Avoid unnecessary repeated requests and repeated translations. Waiting and failures say why

## UI/UX requirements

- Clean, warm, lived-in. Like an open cookbook, not an admin dashboard
- Content pages are single-column full-width section cards: find a dish and detail each form a card
- Primary actions use verbs that match the result: "Favorite" and "Add to shopping list" take effect immediately with a hint
- Empty, waiting, and failed states all say the next step. A failed section explains only itself
- Keyboard: visible focus. Escape or outside click closes pop-ups. State is not color-only
- Desktop and mobile: no clipped controls
