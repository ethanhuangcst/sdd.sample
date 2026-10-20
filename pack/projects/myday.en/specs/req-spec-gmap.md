# MyDay Web App

- A daily planning web app that helps users start the day with personalized recommendations.
- This requirement uses Google Maps only for geocoding, restaurant search, and map links. See `specs/tech-spec.md` from the same copy. Do not configure AMap. Do not call Google Weather. Weather uses Open-Meteo.
- Derive the zodiac sign locally from the birth date. Do not call a horoscope service.
- No saved profiles and no `data/` folder. The photo lives only for this outfit request. Refresh returns to page 1.
- Location is typed only. Do not use browser geolocation.
- Write the UI in English. This demo does not add a second locale.
- Intended for demos outside mainland China. One run reads this requirement only.

## Overview

- Page 1, "Introduce yourself": personal details. Submit only when every field is filled. Stay on page 1 when location cannot be resolved.
- Page 2, "Plan my day": three equal-width sections stacked top to bottom — your status, today's recommendations, where to eat

## Requirements

### Page 1, "Introduce yourself"

Every field is required.

- Name. Used for greeting, for example "Hello, Alex"
- Gender. One of: Male, Female, Prefer not to say
- Birth date. Used for age and zodiac sign. The date must not be after today in the browser. Age uses the computed local calendar date; subtract one year if this year's birthday has not occurred yet. Zodiac uses Gregorian month and day only, not birth time.
- What I am doing today. For example "traveling"
- My mood now. For example "excited"
- My location now. For example "Taipei Shilin". Hint beside the field: "A more precise place gives better recommendations"
- Upload a photo. Used only for the outfit portrait on page 2. Do not write it to disk

#### Submit

- Button label is "Submit"
- If any field is missing, the button is gray and not clickable
- When every field is filled, the button is clickable
- Treat a field as missing if it is empty after trimming leading and trailing whitespace
- On click, run geocoding once through Google Maps
- No result: stay on page 1. Mark "My location now" in red and show text asking the user to enter again. Do not go to page 2
- A result: go to page 2. Pass coordinates, city name, and local time computed on the client for this location. Do not call an external time zone API
- Pass name, gender, birth date, what you are doing, mood, location, and photo together

### Page 2, "Plan my day"

Three sections stack vertically, equal width, each full width of the content column.

When the user reaches page 2, start weather, the status summary, itinerary text, and outfit text together. Start the portrait only after outfit text returns. The where-to-eat section waits until the user enters a cuisine. If one block fails, show the error only in that block. Do not offer a retry.

This page provides "Plan again". On click, return to page 1 and clear name, gender, birth date, what you are doing, mood, location, photo, and everything already loaded on page 2.

#### Your status

- "Hello, {name}"
- Age and zodiac name from the birth date
- City, current weather condition, temperature, and an icon that matches the condition. City name comes from this geocode. Weather follows the coordinate rules in the tech spec and calls Open-Meteo. Display weather labels in English
- If weather fails, show the error only in this section. Still start the status summary, itinerary, and outfit text. In those three texts, weather is "unknown". Do not offer a retry
- One status summary line from the prompt below. No more than 40 words

##### Status summary prompt

Inputs: zodiac, location as the user typed it, what I am doing today, my mood now, and weather. If weather failed, use "unknown".

```text
You are a day-planning assistant. Write one English sentence from the facts below, at most 40 words. Keep the tone witty, light, and specific, as if you are speaking to this person. No lists. No outfit. No fortune-telling. Do not invent places, weather, or restaurant names that are not in the facts.

Zodiac: {sign}
Location: {location}
Doing today: {activity}
Mood: {mood}
Weather: {weather or unknown}
```

#### Today's recommendations

- Itinerary text from the itinerary prompt. One English paragraph, 120 to 180 characters including punctuation. Cover what to eat and what to do. Link food and activities to common traits of the zodiac sign and name the sign once. Do not say where to go. Do not use "food, shelter, travel, fun" as subheadings
- Outfit text from the outfit prompt, 80 to 140 characters. The reader should be able to identify top, bottom, shoes, and one accessory. If this text fails, do not start the portrait and explain why in the portrait area
- Portrait: from outfit text and the page 1 photo, generate a highly realistic full-body image. Face matches the photo; clothes match the outfit text. Center the image horizontally in its area, max width within the content column, keep aspect ratio. If the photo cannot be read, show an error. Never generate without that photo

Local time is the client local time from page 1 for this location. When gender is Prefer not to say, prompts must not ask the model to guess gender.

##### Itinerary prompt

Inputs: zodiac, gender, location (user text), what I am doing today, my mood now, local time, weather. If weather failed, use "unknown".

```text
You are a day-planning assistant. From the facts below, plan from local time through tonight in one natural English paragraph, 120 to 180 characters including punctuation. No bullet list, no subheadings.
Write in this order without breaking flow.
1. Two sentences of empathy: the first summarizes mood, what they are doing, local time, and weather; the second is a short encouraging line that fits the zodiac personality and names the sign.
2. Then what to eat and what to do. You may suggest a cuisine. Food and activities should reflect common traits of the sign without repeating the sign name. Use personality to pick cuisine and how to wrap up the evening. No separate personality essay, no horoscope.
3. Optionally add one more comfort for tonight, still grounded in the facts. No shop names, hours, or outfit. Do not invent facts. When gender is Prefer not to say, do not guess gender; address the user as "you".

Zodiac: {sign}
Gender: {gender}
Location: {location}
Doing today: {activity}
Mood: {mood}
Local time: {local time}
Weather: {weather or unknown}
```

##### Outfit prompt

Inputs: gender, what I am doing today, and weather. If weather failed, use "unknown".

```text
You are an outfit assistant. Write today's outfit in one English paragraph, 80 to 140 characters. Name top, bottom, shoes, and one accessory, with color and material. No bullet list, no itinerary, no brands, no horoscope. When gender is Prefer not to say, the outfit works for any gender. When weather is unknown, do not guess the temperature.

Gender: {gender}
Doing today: {activity}
Weather: {weather or unknown}
```

##### Portrait prompt

Insert outfit text into the template below and attach the page 1 photo.

```text
From the outfit below, generate a highly realistic full-body photo of this person. Face and hair match the attached photo; do not replace the person. Clothes, shoes, and accessories follow the outfit text; do not redesign them. Only this person in frame; no text, watermark, or logos. Never generate without the photo.
{outfit text}
```

#### Where to eat

- The user enters a cuisine as text. Do not search when the trimmed text is empty
- Use the page 1 location. Search with Google Maps only. Radius 5000 m, one search. Sort by distance, nearest first
- Each row shows restaurant name, distance, and address. Clicking a row opens Google Maps in a new tab
- Do not show rating, review count, or review snippets. No overlay dialog, no per-row detail API
- When there are no results, tell the user they can change cuisine or tap "Plan again". Never show a blank list

## 3. Tests

### Component tests

Both use fixtures and do not call the network. Do not pick a photo. Do not assert coordinates.

- When location cannot be resolved, stay on page 1 and show text asking the user to enter again
- After "Plan again", return to page 1 with empty fields

### Acceptance test

The test photo is `specs/test-img.png`. It is a 200×200 PNG of a person, not 1×1. Component tests do not use it. When the file picker cannot be used, send this image as the photo for that request and continue. Do not change the upload implementation for this. Do not write the photo to disk.

After both fixture tests pass, run `npm run dev` in `web/`. Follow the path below once in the browser. When it passes, give the local URL in the reply and stop.

- Open "Introduce yourself". With every field empty, "Submit" is gray and not clickable.
- Fill name, gender, birth date, what you are doing today, mood, a photo, and "Taipei Shilin". Use `specs/test-img.png` for the photo.
- Click "Submit" once. Enter "Plan my day". You should see a hello with the name, plus age, zodiac, city, weather condition, temperature, weather icon, one status summary line, itinerary, outfit text, and outfit portrait.
- No restaurant search before a cuisine is entered. After one cuisine, the restaurant list appears. Each row has name, distance, and address. Clicking a row opens Google Maps in a new tab.
- Stop after this one pass. Do not change layout. Re-run the two fixture tests only if this pass changed code. "Plan again" is covered by fixtures; do not click it in this pass.

## 4. UI and UX

The UI should feel clean, casual, light, and fun. Choose palette, typography, spacing, and motion during implementation.

- Page 2 stacks three sections vertically. Each has a title and its own loading, empty, and error state
- Restaurants are a vertical list, one row per place. Do not use a multi-column card grid
- Place the portrait inside "Today's recommendations", centered horizontally, max width within the content column, keep aspect ratio. Do not stretch or crop to a tiny icon. The full body should be visible on desktop and mobile. While the image is loading, show a loading state, not a broken image. Alt text is "Outfit portrait"
- Page 1 primary action is "Submit". Page 2 offers "Plan again"
- Location errors use both color and text. Empty, loading, and failure states say what to do next. Failure stays in that section only. Never show the word "Timeout"
- Write the UI in English. This demo does not add a second locale
- Focus is visible. Restaurant rows are links that open in a new tab
- Works on desktop and mobile; controls are not clipped. Respect `prefers-reduced-motion` when motion is used
