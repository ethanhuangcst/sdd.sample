# What2Eat — Business requirements

- What2Eat is a web app that helps someone who cannot decide what to eat.
- It uses the person's location, party size, saved preferences, craving, constraints, weather, and time of day. AI and Google Maps pick one nearby place.
- It does not place orders, take payment, or replace a restaurant menu, allergen label, or medical diet advice.

## Who it is for

People who want a quick or random meal pick: one person, a small group, an office lunch, a traveler in a new city, or a household with different diets.

## Features

### Data definitions

#### User

- Username
- Email
- Password
- Avatar
- Preference profile

#### Preference profile

- Cuisines they like
- Cuisines they dislike
- Spice tolerance
- Default party size
- Hard constraints, as free text, such as vegetarian, no pork, or nut allergy

#### Cuisine

- The first version stores the text the user types. It does not keep a cuisine table.
- A Google search uses one Places API (New) Table A type, or `textQuery`. See "Build the shortlist".

### Account and sign-in

#### Register

##### Account

- Username, required. 2 to 32 characters. Letters, digits, spaces, and Chinese characters are allowed.
- Email, required. A valid email. It is the sign-in account after registration.
- Password, required. At least 8 characters, with at least one letter and one digit.
- Avatar, optional. One uploaded image. A missing upload uses the default avatar.
- Default place, required. A city or area. "What to eat?" fills "Search place" with this value.

##### Preferences

- Cuisines they like: many choices, optional
- Cuisines they dislike: many choices, optional
- Spice tolerance: one choice, optional. None, mild, medium, or hot.
- Default party size: optional. A positive integer. Default 1.
- Hard constraints: many items, optional. Free text, such as vegetarian, no pork, or nut allergy.

##### Save

- A failed check stays on this page, marks the failed fields, and writes nothing.
- A passed check saves the user and the preference profile.
- Success opens "What to eat?".

#### Sign in

- The user signs in with email and password.
- A wrong email or a wrong password uses the same message. The message does not say which one failed.
- Success opens "What to eat?".

#### Forgot password

- "Forgot password?" on the sign-in page opens the forgot-password page.
- The user submits the registration email. The app sends a reset email.
- An unknown email and a registered email use the same success message. The message does not reveal whether the email is registered.
- The link in the email opens the reset-password page. After the user sets a new password, that password can sign in.

#### Signed-in header

- Top right of a signed-in page.
- Left to right: the greeting "Hello, {username}", the avatar, and a "Log out" button.
- The username opens "Edit profile".

#### Edit profile

- Fields and checks match "Account" and "Preferences" under "Register".
- A successful save returns to "What to eat?". The header shows the new username and avatar.

### What to eat?

#### Filters

The user enters these on this page:

- "Search place": typed by the user. It starts as the default place from the preference profile. Required.
- A switch between "Exact match" and "Surprise me". The default is "Exact match".
- Filters under "Exact match" are all optional:
  - "Meal context": office lunch, dining alone, date, family meal, out of town, quick bite, or lighter meal.
  - "Party size": starts as the default party size from the preference profile. The user can change it.
  - "What do you want?": a multi-value text field. It starts with the liked cuisines from the preference profile. The user can remove a chip or type a new one.
  - "What to skip?": a multi-value text field. It starts with the disliked cuisines and the hard constraints from the preference profile. The user can remove a chip or type a new one.
- "Surprise me":
  - Choosing "Surprise me" hides every "Exact match" filter.
  - Switching back to "Exact match" keeps the values the user already entered.
  - "Recommend" in "Surprise me" treats "Meal context", "Party size", and "What do you want?" on this page as empty. The baseline search and "Best match" still run. "Best match" uses only the hard constraints from the preference profile.

#### Recommend

- Step 1, "Place and time". Checking the place and reading the local time start together. The time read does not wait for the place check.
  - Check that the typed place can be resolved. If it cannot, mark the field and stay on this page.
  - Use the local time at that place as the search time.
- Step 2, "Build the shortlist". Search only. Do not show the list yet.
- Step 3, "Best match". Choose only from the shortlist. Do not call Google Maps again.

#### Build the shortlist

The server calls Places API (New). It does not call AMap. It does not send `keywords`, `types`, or an AMap type code such as `050000`.

Coordinates, radius, the API key, and the field mask are written by the server. The model does not output them.

Geocoding uses `GET ${GOOGLE_MAPS_GEOCODE_URL}?address={address}&key=`. Success is `status` `OK`. Coordinates are WGS-84. The circle center is `{ "latitude", "longitude" }`, not longitude then latitude.

Both searches use the same center and a radius of 10000 meters. The radius does not grow. A circle is required. Omitting `circle` returns 400.

Headers for both searches: `Content-Type: application/json`, `X-Goog-Api-Key`, and `X-Goog-FieldMask`. The key stays on the server. No `NEXT_PUBLIC_*` map key.

Field mask for this product:

```text
places.id,places.displayName,places.formattedAddress,places.location,places.primaryTypeDisplayName,places.currentOpeningHours,places.regularOpeningHours,places.rating,places.priceLevel
```

- Baseline: one `POST` to `GOOGLE_MAPS_PLACES_SEARCH_NEARBY_URL` (`places:searchNearby`). No text query. Nearby Search does not accept a text query. `includedTypes` is `["restaurant"]`. That is the dining filter for this product. `maxResultCount` is 20, the Google maximum (1 to 20). The area is `locationRestriction.circle`. "Surprise me" makes only this call.
- "Exact match" with non-blank "What do you want?" after trim: one more `POST` to `GOOGLE_MAPS_PLACES_SEARCH_TEXT_URL` (`places:searchText`). Merge with the baseline by `places.id`. If "What do you want?" is empty, do not call the model.
- The map-query prompt returns only `textQuery` and `includedType`.
  - `textQuery`: one string, at most 80 characters, taken only from "What do you want?". No city, address, place name, or coordinates. A place name in `textQuery` can make Google ignore `locationBias`.
  - `includedType`: one Table A type from Food and Drink. Text Search accepts one type, not a list and not a `|` joined string. Table B values, including `food`, are not filters. If the words do not match a Table A Food and Drink type, `includedType` is `restaurant` and the words stay in `textQuery`.
- The server builds the Text Search body. It sets `strictTypeFiltering` to `true`, `pageSize` to 20, `languageCode` to `en`, and `locationBias.circle` from the geocode result. It does not send `openNow`. It does not follow `nextPageToken`. It does not send `locationRestriction` on this call: Text Search `locationRestriction` is a rectangle, and this pack uses the circle form in `pack/tech-spec.en/tech-spec.md`.
- After Text Search returns, drop any place farther than 10000 meters from the center. `locationBias` can include places outside the circle. Do not search again with a larger radius.
- Do not put the search time in the request. Compare `currentOpeningHours` with the local time at the place. If those hours are missing, use `regularOpeningHours`. If both are missing, the label is "Hours uncertain". Do not label that place as open.
- One Nearby Search returns at most 20 places. One Text Search page returns at most 20. The merged list is the union of those two calls. This replaces the AMap cap of 200. Google documents Nearby Search `maxResultCount` as 1 to 20, and Text Search as at most 20 results on a page.
- Empty-result reasons:
  - The baseline returns no places: no places nearby.
  - The baseline has places, and the merge or the later exclusions leave none: filters are too narrow.
  - The map call fails: provider failure. Saved favorites still show.
- Prompt:

```text
From "What do you want?" only, output JSON: {"textQuery":"one query","includedType":"one Table A Food and Drink type, or restaurant"}.
textQuery is at most 80 characters. No city, address, place name, or coordinates.
includedType is one Places API (New) Table A Food and Drink type. One type only. Not a list. Not AMap codes. Not a string joined by |.
If the words do not match a Table A Food and Drink type, set includedType to restaurant and keep the words in textQuery.
Do not output latitude, longitude, radius, a field mask, an API key, opening hours, or any other field.
```

#### Best match

- Rules remove blacklist places first, then places that conflict with "What to skip?" and the hard constraints. Exact match uses "What to skip?" on this page. Surprise me uses the hard constraints on the preference profile. Allergies and foods to avoid are not decided by the model.
- The recommend prompt then picks a short list from the places that remain. The model may return only those `places.id` values, plus a match reason. The reason is labeled as a model suggestion. It is not a restaurant promise or diet advice.
- Prompt inputs: meal context, party size, spice tolerance from the preference profile, what to skip, hard constraints, and the place facts that remain. In Surprise me, meal context, party size, and "What do you want?" are empty.
- Prompt:

```text
Pick a short list only from the given place ids. Each item has a place id and a match reason.
The match reason uses only meal context, party size, spice tolerance, what to skip, and hard constraints, and it says it is a model suggestion.
Do not return a place that is not in the list. Do not invent an address, hours, price, or phone number.
```

#### Recommended places

- Shown under the filters. This area stays hidden until the user chooses "Recommend".
- After the shortlist and best match finish, show the best-match short list in a grid.
- Each card shows the name (`displayName`), cuisine tag (`primaryTypeDisplayName`), distance, open status, and match reason. Distance is computed from the search center and `places.location`. A missing field uses a placeholder.
- The same constraints can be shuffled again. A shuffle only reorders or resamples. It does not call Google Maps again.
- When there is no best match, show the empty-result reason from the shortlist.

#### Place details

- A dialog shows the place. Close, Escape, or a click on the backdrop closes it.
- Show address, hours, rating, price level, and other returned fields. A missing field uses a placeholder.
- "Save" and "Blacklist" take effect only after the user confirms.

#### Saved places and history

- Save a liked place. Hide a place the user skips again and again for a cooling-off period.
- Show recent decisions: what was chosen, and whether the user marked "went" or "skipped".
- A past decision can be run again with the current place status.

#### Blacklist

- From place details, the user can blacklist the current place. Confirm first.
- Each row stores: user id, Google `places.id`, a name snapshot, and the time it was added. The snapshot is for the list only. Matching uses `places.id`.
- After that, the place does not appear in the recommended list or in a shuffle. The shortlist may still return it. The rule step removes it.
- If the place is saved, blacklisting it also removes the save.
- The blacklist page lists rows from newest to oldest. The user can remove a row. After removal, a later recommendation may show that place again.
- A blacklist row is one place. "What to skip?" is text on this session or on the profile. They do not replace each other.

### Data quality

- Always show when the place facts were last updated.
- Mark stale, incomplete, unavailable, or conflicting data. Keep user preference, place facts, and model suggestions distinct.
- An AI recommendation is not a restaurant promise and not health guidance.
- One provider failure does not take down the whole app.

### Performance

- While a nearby search is loading, show saved favorites that are already known.
- Meet the latency targets for decide-now, search, details, and AI responses.
- A cache entry includes its last-updated time. Cached open or closed status is not labeled as guaranteed live status.
- Use timeouts, retries, and fallbacks. Do not repeat an API or LLM call that is not needed.
