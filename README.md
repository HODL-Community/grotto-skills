# Grotto Skills

Public skill repository for The Grotto developer ecosystem.

This repo is intentionally simple: each skill lives under `skills/<skill-name>/` with a `SKILL.md` entry point plus optional `templates/`, `references/`, and `assets/`.

## Included skills

### Grotto Game Runtime Developer SDK

Path: `skills/grotto-game-runtime-developer-sdk/SKILL.md`

Use this skill when building Grotto-hosted HTML5 or WebGL games that need:

- trusted Grotto player identity
- cloud saves and autosave
- save slots and conflict-safe sync
- leaderboards and gameplay events
- presence and future multiplayer hooks
- Railway or Supabase-backed custom game services

Public SDK URL:

```html
<script src="https://api.enterthegrotto.xyz/sdk/grotto-game-runtime.v1.js"></script>
```

Public Grotto skills page:

https://www.enterthegrotto.xyz/skills

Live Grotto API docs:

https://api.enterthegrotto.xyz/docs

This repo is the canonical public source for skill markdown. Website download links should point at the raw files in this repo instead of maintaining separate markdown copies.

## Repository layout

```text
skills/
  grotto-game-runtime-developer-sdk/
    SKILL.md
    references/
      sdk-contract.md
    templates/
      minimal-runtime-game.html
assets/
  grotto-game-runtime-developer-sdk/
    grotto-runtime-sdk-ad-redo.png
relevance/
  manifest.json
```

## Contributing

Open a pull request to add or update a skill. Keep skills creator-facing, practical, and free of private credentials.

### Safety rules

- Do not commit API keys, bearer tokens, Privy secrets, Railway/Vercel tokens, `gst_*`, or `grs_*` values.
- Redact secrets as `[REDACTED]`.
- Prefer runnable examples and explicit security notes.
- Keep Grotto identity/session guidance clear: games should not trust wallet/user IDs supplied by client-side code.

## PR watch

This repo is watched for pull requests by Bob/Hermes so skill contributions can be triaged quickly.
