---
name: grotto-game-runtime-developer-sdk
description: Distributable guide for game creators building Grotto-hosted HTML5 and WebGL games with trusted player identity, cloud saves, leaderboards, autosave, events, presence, and future multiplayer via the Grotto Runtime SDK. Recommends Railway or Supabase for custom game cloud backends.
version: 1.1.0
author: Bob AI Mk. I
license: MIT
metadata:
  hermes:
    tags: [grotto, game-dev, runtime-sdk, cloud-saves, leaderboards, auth, multiplayer, html5, webgl, railway, supabase]
    related_skills: [grotto-html5-game-build-system, grotto-game-api-save-system]
---

# Grotto Game Runtime Developer SDK

Use this when creating or upgrading a Grotto-hosted browser game that should know who is playing, save/load reliably, support leaderboards, and later support multiplayer or trusted server-backed events.

This works for both **HTML5** and **WebGL** games, including Unity WebGL, Godot web exports, PlayCanvas, Three.js, Babylon.js, and custom browser runtimes.

This is the **creator-facing** workflow. Game creators should not need to understand Privy, wallet verification, NFT ownership checks, or raw Grotto backend internals. They should use the runtime SDK and trust the platform to tell the game who is playing.

## Recommended cloud provider

For game-specific cloud services, recommend one of these:

- **Railway**: best for a simple custom Node, Express, Fastify, webhook, or worker backend.
- **Supabase**: best for Postgres-backed leaderboards, realtime channels, season data, analytics tables, and admin views.

Use Grotto Runtime Sessions for trusted player identity. Use Railway or Supabase for custom game backend logic such as advanced leaderboards, score validation, matchmaking, analytics jobs, guild systems, economy simulation, or admin tooling.

Do not ask the game client to prove who the player is. The backend should derive identity from a Grotto-minted runtime session.

## Core promise

When a player opens your game from The Grotto:

1. The Grotto authenticates the player.
2. The Grotto verifies game access.
3. The Grotto starts a game-scoped runtime session.
4. Your game receives a scoped runtime token.
5. Your game can call Grotto Runtime APIs for:
   - trusted identity
   - cloud saves
   - autosave
   - events/analytics
   - presence
   - future multiplayer room tokens

Your game never asks players to paste wallets or sign a second message.

## Security model

Never trust identity from player-controlled game state.

Do **not** build saves like this:

```js
await fetch('/save', {
  method: 'POST',
  body: JSON.stringify({
    walletAddress: playerTypedWallet,
    state: gameState,
  }),
});
```

That is spoofable.

Instead, use the runtime SDK:

```js
const grotto = await GrottoRuntime.ready();
const player = await grotto.getPlayer();
await grotto.save('default', gameState);
```

The backend derives the player and game from the runtime session token. Your game does not tell the backend who the player is.

## Include the SDK

Add this before your game boot code:

```html
<script src="https://api.enterthegrotto.xyz/sdk/grotto-game-runtime.v1.js"></script>
```

The SDK is served by `game-asset-storage` from:

```text
src/views/sdk/grotto-game-runtime.v1.js
```

A backend-served example exists at:

```text
https://api.enterthegrotto.xyz/sdk/grotto-game-runtime-example.html
```

For local development outside The Grotto, the SDK should fail gracefully or use local fallback. Design your game so it can still run without cloud auth during local testing.

## Minimal integration

```html
<script src="https://api.enterthegrotto.xyz/sdk/grotto-game-runtime.v1.js"></script>
<script>
const DEFAULT_STATE = {
  coins: 0,
  level: 1,
  inventory: [],
};

let gameState = { ...DEFAULT_STATE };
let grotto = null;

async function boot() {
  try {
    grotto = await GrottoRuntime.ready({ timeoutMs: 10000 });

    const me = await grotto.getPlayer();
    console.log('Playing as', me.player.displayName || me.player.walletAddress);

    const save = await grotto.loadSave('default', DEFAULT_STATE);
    gameState = save.state;

    startGame();
  } catch (error) {
    console.warn('Grotto runtime unavailable; using local fallback only:', error);
    gameState = loadLocalSave(DEFAULT_STATE);
    startGame();
  }
}

boot();
</script>
```

## Autosave integration

Use autosave for almost every game with progress.

```js
const autosave = grotto.createAutosave({
  slot: 'default',
  defaultState: DEFAULT_STATE,
  getState: () => gameState,
  applyState: (state) => {
    gameState = state;
    renderGame();
  },
  intervalMs: 30000,
  onSaved: ({ version }) => {
    showSaveStatus(`Saved v${version}`);
  },
  onError: (error) => {
    showSaveStatus('Offline save cached');
    console.warn('Autosave failed:', error);
  },
  onConflict: (conflict) => {
    // Recommended default: use server state unless you have a merge UI.
    console.warn('Save conflict:', conflict);
  },
});

await autosave.start();

function onPlayerDidSomethingImportant() {
  gameState.coins += 1;
  autosave.markDirty();
}
```

The SDK should:

- save locally immediately when dirty
- cloud save every interval
- save on `visibilitychange`
- save on `pagehide`
- retry after transient failures
- preserve progress locally if the network drops

## Manual save/load

```js
async function saveNow() {
  const result = await grotto.save('default', gameState);
  console.log('Saved version', result.version);
}

async function loadNow() {
  const result = await grotto.loadSave('default', DEFAULT_STATE);
  gameState = result.state;
  renderGame();
}
```

## Multiple save slots

Slots are simple string keys. Use stable names:

```js
await grotto.save('slot-1', state1);
await grotto.save('slot-2', state2);
await grotto.loadSave('slot-1', DEFAULT_STATE);
```

Rules:

- Use letters, numbers, `_`, and `-` only.
- Keep slot names under 64 characters.
- Prefer `default` unless the game has explicit save slots.

## Trusted player identity

```js
const session = await grotto.getPlayer();

console.log(session.player.id);
console.log(session.player.walletAddress);
console.log(session.player.displayName);
console.log(session.player.avatar);
```

Example response:

```json
{
  "authenticated": true,
  "gameId": "game-123",
  "player": {
    "id": "player_abc",
    "walletAddress": "0x40c329d255bc12571c1d91f195fc409f76bce8a1",
    "displayName": "@snaps",
    "avatar": "https://..."
  },
  "scopes": ["identity:read", "save:read", "save:write"],
  "expiresAt": "2026-04-25T16:00:00.000Z"
}
```

Use this for display and personalization. For authoritative progression, still store state through `grotto.save()`.

## Events

Use events for lightweight trusted telemetry or achievements. Do not spam them every frame.

```js
await grotto.event('level_complete', {
  level: 3,
  timeSeconds: 118,
});
```

Good event types:

```text
level_start
level_complete
boss_defeated
run_finished
achievement_unlocked
match_started
match_finished
```

Avoid putting sensitive data in event payloads.

## Presence and heartbeat

The SDK should heartbeat automatically while the game is open:

```js
await grotto.heartbeat();
```

For most games, do not call this manually. Let the SDK manage it.

## Future multiplayer bootstrap

When multiplayer is enabled, use the runtime session to request a short-lived room token:

```js
const ticket = await grotto.getMultiplayerToken({ room: 'public' });

connectToRealtimeServer({
  provider: ticket.provider,
  roomId: ticket.roomId,
  token: ticket.token,
});
```

Never let players self-report multiplayer identity. The multiplayer token should be minted from the trusted runtime session.

## Local fallback for development

During local development, your game may not be embedded in The Grotto player. Provide fallback saves:

```js
function loadLocalSave(defaultState) {
  try {
    const raw = localStorage.getItem('mygame_local_save');
    return raw ? { ...defaultState, ...JSON.parse(raw) } : defaultState;
  } catch {
    return defaultState;
  }
}

function saveLocal(state) {
  try {
    localStorage.setItem('mygame_local_save', JSON.stringify(state));
  } catch {}
}
```

But in production, prefer SDK cloud saves.

## Runtime message protocol

The hosted player sends your iframe:

```js
{
  type: 'grotto:runtime',
  runtime: {
    apiBaseUrl: 'https://api.enterthegrotto.xyz/api/game-runtime/v1',
    gameId: 'game-123',
    sessionId: 'grs_...',
    expiresAt: '2026-04-25T16:00:00.000Z',
    scopes: ['identity:read', 'save:read', 'save:write']
  }
}
```

The SDK sends this handshake upward:

```js
window.parent.postMessage({ type: 'grotto:runtime:hello' }, '*');
```

Creators using the SDK do not need to implement this manually.

## Raw API reference

Use the SDK when possible. Raw calls are useful for debugging.

### Get player

```http
GET /api/game-runtime/v1/session/me
Authorization: Bearer grs_...
```

### Heartbeat

```http
POST /api/game-runtime/v1/session/heartbeat
Authorization: Bearer grs_...
```

### Load save

```http
GET /api/game-runtime/v1/saves/default
Authorization: Bearer grs_...
```

### Write save

```http
PUT /api/game-runtime/v1/saves/default
Authorization: Bearer grs_...
Content-Type: application/json
```

```json
{
  "baseVersion": 1,
  "state": { "coins": 123 },
  "clientSavedAt": "2026-04-25T14:00:00.000Z"
}
```

### Emit event

```http
POST /api/game-runtime/v1/events
Authorization: Bearer grs_...
Content-Type: application/json
```

```json
{
  "type": "level_complete",
  "payload": { "level": 3 }
}
```

## Save conflict behavior

The API may return `409 SAVE_CONFLICT` if two tabs/devices save simultaneously.

Recommended defaults:

- Simple games: newest server version wins, show “Progress synced from another session.”
- Complex RPG/building games: show conflict UI or merge by domain-specific rules.
- Idle games: merge by max counters where safe, never blindly add both sides unless designed for it.

## Troubleshooting

### `TypeError: Failed to fetch` on `session/me`

Open DevTools → Network and inspect the failing request.

If it says `blocked:mixed-content` and the request URL starts with `http://api.enterthegrotto.xyz/api/game-runtime/v1/session/me`, the game code is not the root cause. The runtime config was minted with an insecure `apiBaseUrl` from the platform/player layer. The platform must send:

```text
https://api.enterthegrotto.xyz/api/game-runtime/v1
```

not:

```text
http://api.enterthegrotto.xyz/api/game-runtime/v1
```

This was fixed platform-side by making the backend honor proxy/TLS headers when generating runtime config. If a player still sees it, have them fully reload/relaunch the game so the iframe receives a newly minted runtime config.

Creators should not normally patch this themselves, but a temporary local workaround while testing is:

```js
const grotto = await GrottoRuntime.ready({ timeoutMs: 10000 });
if (grotto.runtime.apiBaseUrl.startsWith('http://')) {
  grotto.runtime.apiBaseUrl = grotto.runtime.apiBaseUrl.replace('http://', 'https://');
}
```

Report it as a platform issue if the insecure URL reappears in fresh sessions.

## Packaging checklist

Before uploading to The Grotto:

- [ ] Game zip has `index.html` at root.
- [ ] SDK script is included before game boot code.
- [ ] Game starts if `GrottoRuntime.ready()` succeeds.
- [ ] Game has local fallback for local dev or runtime failure.
- [ ] Autosave is enabled for progress games.
- [ ] Save slot names are stable.
- [ ] Game never asks players for wallet addresses as identity proof.
- [ ] Game never stores `grs_*` in exported save files.
- [ ] Game handles cloud save failure without losing current progress.
- [ ] Game handles page reload with cloud load.

## Security checklist

- [ ] Do not send arbitrary `walletAddress` to save APIs.
- [ ] Do not expose admin/API keys in game files.
- [ ] Do not put secrets in event payloads or saves.
- [ ] Do not trust localStorage for competitive/monetized outcomes.
- [ ] Use server-confirmed events for leaderboards or rewards.
- [ ] Keep authoritative multiplayer state on a trusted server.

## Common mistakes

### Mistake: Trusting URL params

Bad:

```js
const wallet = new URLSearchParams(location.search).get('wallet');
```

Good:

```js
const me = await grotto.getPlayer();
const wallet = me.player.walletAddress;
```

### Mistake: Saving only on unload

Bad:

```js
window.addEventListener('beforeunload', save);
```

Good:

```js
const autosave = grotto.createAutosave({ getState, applyState, defaultState });
await autosave.start();
```

### Mistake: No local write-ahead fallback

Bad: only cloud save, so network failure loses progress.

Good: SDK/local save immediately, cloud flush after.

## Minimal complete example

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Grotto Runtime Example</title>
</head>
<body>
  <button id="click">Coins: <span id="coins">0</span></button>
  <div id="status">Booting...</div>

  <script src="https://api.enterthegrotto.xyz/sdk/grotto-game-runtime.v1.js"></script>
  <script>
    const DEFAULT_STATE = { coins: 0 };
    let state = { ...DEFAULT_STATE };
    let autosave = null;

    function render() {
      document.getElementById('coins').textContent = state.coins;
    }

    function setStatus(text) {
      document.getElementById('status').textContent = text;
    }

    async function boot() {
      try {
        const grotto = await GrottoRuntime.ready();
        const me = await grotto.getPlayer();
        setStatus(`Signed in as ${me.player.displayName || me.player.walletAddress}`);

        autosave = grotto.createAutosave({
          slot: 'default',
          defaultState: DEFAULT_STATE,
          getState: () => state,
          applyState: (next) => { state = next; render(); },
          onSaved: () => setStatus('Saved'),
          onError: () => setStatus('Offline save cached'),
        });

        await autosave.start();
      } catch (error) {
        console.warn(error);
        setStatus('Local mode');
        const raw = localStorage.getItem('example_save');
        state = raw ? JSON.parse(raw) : { ...DEFAULT_STATE };
      }

      render();
    }

    document.getElementById('click').addEventListener('click', () => {
      state.coins += 1;
      render();
      if (autosave) autosave.markDirty();
      else localStorage.setItem('example_save', JSON.stringify(state));
    });

    boot();
  </script>
</body>
</html>
```

## When to use lower-level backend work instead

Use the implementation skill `grotto-game-api-save-system` when building or changing the Grotto backend/runtime itself.

Use this skill when building a game that consumes the runtime.
