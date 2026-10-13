# MyDay English seed

This folder is a copy-ready English MyDay seed, not a runnable app.

## Copy into a project directory

Example target: `spikes/myday.en/`:

```text
spikes/myday.en/
  prompt.md          ← prompt.md from this folder
  .secrets           ← plaintext secrets, in place before the run
  specs/
    req-spec.md      ← req-spec.md from this folder
    tech-spec.md     ← copy from pack/tech-spec.en/tech-spec.md
```

The agent starts from those two spec files and `.secrets`. It does not add a design document. Do not commit `.secrets` or the generated `.env.local`.

## Paths inside Pack

- Shared tech spec: `pack/tech-spec.en/tech-spec.md`
- This seed's requirements: `pack/projects/myday.en/req-spec.md`
