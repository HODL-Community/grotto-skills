---
name: grotto-hosted-game-github-workflow
description: Maintain Grotto games with GitHub version control, CI tests, hosted Railway/Vercel clients, and a small Grotto iframe wrapper for fast updates and rollback.
version: 1.0.0
author: Bob AI Mk. I
license: MIT
metadata:
  hermes:
    tags: [grotto, game-dev, github, version-control, testing, ci, railway, vercel, iframe, wrapper]
    related_skills: [grotto-game-runtime-developer-sdk]
---

# Grotto Hosted Game GitHub Workflow

Use when a Grotto game should update like a normal web app: GitHub PRs, CI tests, auto-deploys, previews, and rollback.

## Pattern

1. Real game client lives in GitHub.
2. Client deploys to Railway/Vercel/Netlify/Cloudflare Pages over HTTPS.
3. Host auto-deploys from `main` or release tags.
4. Grotto upload is only a tiny `index.html` wrapper.
5. Wrapper iframes the hosted client and forwards Grotto runtime messages.

Benefits: fast updates, PR review, test gates, preview deploys, rollback, and fewer full zip uploads to Grotto.

## Repo layout

```text
my-grotto-game/
  package.json
  src/
  public/
  tests/
  wrapper/index.html
```

## Hosted client requirements

```html
<script src="https://api.enterthegrotto.xyz/sdk/grotto-game-runtime.v1.js"></script>
```

```js
const grotto = await GrottoRuntime.ready({ timeoutMs: 10000 });
const player = await grotto.getPlayer();
const save = await grotto.loadSave('default', DEFAULT_STATE);
startGame({ player, state: save.state });
```

The hosted URL must be HTTPS.

## Grotto wrapper

Upload this as the root `index.html` in the Grotto game zip.

```html
<!doctype html><html><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>My Grotto Game</title>
<style>html,body,#game{width:100%;height:100%;margin:0;border:0;overflow:hidden;background:#050510}</style>
</head><body>
<iframe id="game" title="My Grotto Game" sandbox="allow-scripts allow-same-origin allow-forms allow-pointer-lock allow-popups" allow="fullscreen; autoplay; clipboard-read; clipboard-write; gamepad"></iframe>
<script>
const REMOTE_GAME_URL = 'https://my-game.up.railway.app';
const iframe = document.getElementById('game');
const params = new URLSearchParams(location.search);
params.set('embedded', 'grotto');
iframe.src = `${REMOTE_GAME_URL}/?${params}`;
addEventListener('message', (event) => {
  const msg = event.data;
  if (!msg || typeof msg !== 'object') return;
  if (msg.type === 'grotto:runtime:hello' && event.source === iframe.contentWindow) {
    parent.postMessage({ type: 'grotto:runtime:hello' }, '*');
  }
  if (msg.type === 'grotto:runtime') {
    iframe.contentWindow?.postMessage(msg, REMOTE_GAME_URL);
  }
});
</script></body></html>
```

## Maintenance loop

1. Branch in GitHub.
2. Change game client.
3. Run tests/build locally.
4. Open PR.
5. CI runs unit/lint/build/Playwright checks.
6. Merge to `main`.
7. Railway/Vercel auto-deploys.
8. Existing Grotto wrapper loads new client.
9. Roll back by reverting Git or rolling back deploy.

## Minimal CI

```yaml
name: game-client-ci
on: [pull_request]
jobs:
  test-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm test -- --run
      - run: npm run build
```

## Wrapper regression test

```js
import { readFileSync } from 'node:fs';
import { describe, expect, it } from 'vitest';
const wrapper = readFileSync('wrapper/index.html', 'utf8');
describe('Grotto wrapper', () => {
  it('embeds hosted client and forwards runtime', () => {
    expect(wrapper).toContain('https://my-game.up.railway.app');
    expect(wrapper).toContain('grotto:runtime:hello');
    expect(wrapper).toContain('grotto:runtime');
    expect(wrapper).toContain('postMessage');
  });
});
```

## Do not use when

- game must be a permanent self-contained archive
- hosted domain is not durable
- third-party scripts are unstable
- wrapper cannot forward runtime messages due sandbox/CSP
