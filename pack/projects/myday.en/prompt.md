# Restrictions

- Execute only the prompt below. Do not create a Makefile. Do not use i18n. Do not ask whether the result is usable before you finish.
- I am working on other tasks and cannot be disturbed.
- Run automatically; only stop when blocked.
- Work in the current project root. Do not reference other projects. Do not write code or docs outside this project.

# Setup

1. Confirm `specs/req-spec.md`, `specs/tech-spec.md`, and `specs/.secrets` are present. If any one is missing, that is a block.
2. Read these three files.
3. Read the plaintext secrets from `specs/.secrets` and write the variables you use into `.env.local`. If a variable this project needs is empty, that is a block.
4. If there is no runnable Next.js app yet, scaffold a minimal app from the install list in Non-functional requirements in `specs/req-spec.md`. Use a non-interactive command. Do not stop on the scaffold prompts. Do not move the app because the directory is not empty.
5. Install dependencies with the npm registry in `specs/tech-spec.md` that matches the current location.
6. Send one short request for each capability named in `specs/req-spec.md`, except image. Image is used on page 4. If one of them fails, that is a block.

# Build

- Build page 1, page 2, page 3, then page 4, in that order from `specs/req-spec.md`.
- Pages, waiting, failure, and tests follow the requirements.
- How to call each capability follows `specs/tech-spec.md`.

# Done

Complete the "Acceptance test" section in `specs/req-spec.md`.
