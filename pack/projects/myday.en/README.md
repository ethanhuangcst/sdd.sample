# MyDay English seed

This folder is a copy-ready English MyDay seed, not a runnable app.

## Copy into a project directory

Example target: `spikes/myday.en/`:

```text
spikes/myday.en/
  prompt.md          ← prompt.md from this folder
  specs/
    req-spec.md      ← req-spec.md from this folder
    tech-spec.md     ← copy from pack/tech-spec.en/tech-spec.md
    .secrets         ← copy from .secrets.example in this folder, then fill in the key
```

The agent starts from the requirements, the tech spec, and `.secrets` in `specs/`. Do not commit `.secrets` or the generated `.env.local`. `.secrets.example` can be committed.

## Paths inside Pack

- Shared tech spec: `pack/tech-spec.en/tech-spec.md`
- This seed's requirements: `pack/projects/myday.en/req-spec.md`
