# MyPoke — Business Requirements

- MyPoke is a Pokémon Trading Card Game (TCG) collection and card-lookup helper. It helps collectors look up cards, manage their collection, and browse official catalog sets and card details
- It primarily serves Pokémon TCG collectors who want to organize their collection and look up cards quickly. Parents helping kids manage collections, and local card shop regulars, may also use it
- Card data comes from the official catalog REST API (https://api.tcgdex.net): free, public, and keyless. Access details are in the shared tech spec, the "Pokémon TCG catalog" section: [`../tech-spec-en.md`](../tech-spec-en.md)

## Features

### User account

- Register a new user (name, email, optional avatar, collector experience level, password)
- Log in with email and password
- Reset the password by email
- Header in the top-right corner of logged-in pages: "Hello, {username}" + avatar + a Logout button

### Home

The home page has four parts: Header, Set overview, Search, and Footer.

#### Header

- Shows the brand name "MyPoke" and navigation
- Navigation offers the "My collection" entry
- When logged out, it shows login and register entries; when logged in, it shows the greeting, avatar, and Logout per "User account"

#### Set overview

- Lists all catalog sets: set name, set symbol, card count, release date (when the catalog has it)
- Clicking a set shows the selected set below that area:
  - Set name, set symbol, release date, card count
  - All cards of the set: catalog thumbnail, card name, number, rarity
  - Each card is marked "owned" or "not owned", based on the "My collection" data
  - Filter by owned / not owned
- Clicking a card opens "Card details"
- The data source and last-updated time are visible on the page

#### Search

- A search box and a submit button; search the official catalog by language, card name, set code, or card number
- Results list matching cards: catalog thumbnail, card name, set, number, rarity
- Different printings of the same card name (language, set, variant) list separately; the user picks the correct one
- Clicking a result opens "Card details"
- When nothing matches, show "No cards found" and say the keywords can be changed

#### Footer

- One fixed line of text; the copy is decided at implementation time

### Card details

- Entry points: a card in the home "Set overview" list, in "Search results", or in "My collection"
- A pop-up or page shows the full catalog details, catalog image, and the data last-updated time; close via Close, Escape, or outside click
- When the card is already in "My collection": a clear "Owned" label plus a "Remove from my collection" button; removal needs confirmation
- When the card is not collected: an "Add to my collection" button; clicking adds directly, no confirmation; on success show the hint "Added to my collection" — not a pop-up; it shows for 3 seconds and fades out
- After adding, quantity defaults to 1 and condition is empty; condition and quantity are edited in "My collection"

### My collection

- Entry: the header navigation
- A responsive grid of collected cards: catalog thumbnail, card name, set, number, rarity, condition (user-entered), quantity
- Filter by set, rarity, and language
- Clicking a card opens "Card details"; clicking a set name returns to the home "Set overview" with that set selected
- Each entry's condition and quantity can be edited

### Data quality

- Always show when catalog information was last updated
- Flag stale, incomplete, unavailable, or conflicting data
- A single provider failure must not take down the whole app

### Performance

- Show stored collection info immediately while catalog data loads
- Meet latency targets for the home page, search, card details, and my collection
- Cache with the last-updated time
- Use timeouts, retries, and fallbacks; avoid unnecessary repeated API calls

## Definition of success

A collector can manage their collection in one place, look up cards quickly, and open card details and set progress (owned vs missing), with the data source and last-updated time visible; the official catalog and the physical card remain the final authority.
