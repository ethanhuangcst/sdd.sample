# MyDay English seed

This folder is a copy-ready English MyDay seed, not a runnable app.

## Copy into a project directory

Example target: `spikes/myday.en/`:

```text
spikes/myday.en/
  specs/
    prompt.md        ← prompt.md from this folder
    req-spec.md      ← req-spec-amap.md or req-spec-gmap.md, one pair only
    tech-spec.md     ← tech-spec-amap.md or tech-spec-gmap.md, matched to req
    .secrets         ← .secrets.amap or .secrets.gmap, renamed
    test-img.png     ← for acceptance when the file picker cannot be used
  web/               ← Next.js app, created by the install steps in tech-spec
```

Agents start from `specs/prompt.md`, the requirements, the tech spec, and `.secrets`. Installation copies `.secrets` into `web/.env.local`. Do not commit `.secrets` or `.env.local`.

## Paths inside Pack

- Mainland China demo: `specs/req-spec-amap.md` with `specs/tech-spec-amap.md`
- Outside mainland China demo: `specs/req-spec-gmap.md` with `specs/tech-spec-gmap.md`
- Copy one pair to `specs/req-spec.md` and `specs/tech-spec.md` per run
- Shared course spec: `pack/tech-spec.en/tech-spec.md`

The slim two-page demo in `specs/` matches the Chinese pack in `pack/projects/myday.cn/specs/`, with English UI and prompts.
