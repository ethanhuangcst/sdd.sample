# Restrictions

- I am working on other tasks and cannot be disturbed.
- Run automatically; only stop when blocked.
- Work in the current project root. Do not reference other projects. Do not write code or docs outside this project.

# Setup

1. Confirm `specs/req-spec.md` and `specs/tech-spec.md` are in the current directory. If either file is missing, stop and ask the user.
2. Read both files:
   - Learn the four pages from `specs/req-spec.md`, and the install list in Non-functional requirements
   - Learn the design principles and the third-party capabilities this project uses from `specs/tech-spec.md`
3. Read plaintext secrets from `.secrets` in the project root. If the file is missing, or a variable this project needs is empty, stop and ask the user. Write the variables you use into `.env.local`. Do not print secrets in the reply, logs, or commits.
4. If there is no runnable Next.js app yet, scaffold a minimal app from the install list in Non-functional requirements in `specs/req-spec.md`. Use a non-interactive command. Do not stop on the scaffold prompts. Use the TypeScript version the scaffold installs.
5. Install dependencies with the npm registry rules in `specs/tech-spec.md`.
6. Verify only the capabilities this project needs. Send one short request each for chat, Open-Meteo, zodiac, and Google geocoding. Do not verify AMap, image generation, Pokémon, or directions. Call image generation on page 4. If a required check fails, stop and ask the user.

# Build

Build page 1, page 2, page 3, then page 4, in that order from `specs/req-spec.md`. On page 3, fortune and weather are fetched only. Do not show them there. Show them on page 4. Section 3.2's "render each block when it finishes" does not apply to showing those two on page 3. After all four pages are written, add one fixture test per page. The tests do not call the network. Page 1: clicking a card enters page 3. Page 2: a missing name does not save. Page 3: both fetches must succeed before page 4. Page 4: weather already fetched is shown, and no restaurant search runs before a cuisine is chosen. This demo does not have to meet the coverage or browser-journey items in section 3.4 of `specs/tech-spec.md`.

Put the technical choices and the UI for each page in code. Do not write a separate design document.

# Done

After tests for all four pages pass, run `npm run dev` and give the local URL in the reply.
