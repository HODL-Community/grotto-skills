---
name: grotto-game-token-gated-inventory
description: Token-gate Grotto game cosmetics, skins, levels, and rewards using Runtime SDK identity, associated wallets, and indexer-backed inventory APIs.
version: 1.0.0
author: Bob AI Mk. I
license: MIT
metadata:
  hermes:
    tags: [grotto, game-dev, token-gating, inventory, indexer, nft, erc1155, erc721, runtime-sdk]
    related_skills: [grotto-game-runtime-developer-sdk]
---

# Grotto Game Token-Gated Inventory

Use when a Grotto-hosted game needs to unlock a skin, item, level, quest, or reward based on NFT/ERC1155/ERC721/game-pass/asset ownership.

## Trust model

1. Get the player from `GrottoRuntime.ready()` and `grotto.getPlayer()`.
2. Resolve all wallets associated with that Grotto account.
3. Query the Grotto API inventory facade for each wallet.
4. Unlock if any associated wallet holds the required contract/token.
5. For valuable rewards, make the final decision server-side.

Do not call raw indexer endpoints from the browser if they require API keys. Do not trust wallet addresses typed by the player or passed in URL params.

## APIs

```text
https://api.enterthegrotto.xyz/docs
GET /api/inventory/:wallet?include_erc721=true&limit=500
GET /api/inventory/games/:wallet
GET /api/inventory/passes/:wallet
```

Inventory groups returned by `/api/inventory/:wallet` may include:

```text
games, gamePasses, assetPacks, assets, nfts, unregistered, summary, pagination
```

## Client-side cosmetic gate tutorial

Requirement: unlock `obsidian-knight` if the player owns token `7` from a specific contract.

```js
const SKIN_ID = 'obsidian-knight';
const GATE = {
  contractAddress: '0x1234567890abcdef1234567890abcdef12345678',
  tokenId: '7',
};

function normalizeWallet(address) {
  const normalized = String(address || '').trim().toLowerCase();
  return /^0x[a-f0-9]{40}$/.test(normalized) ? normalized : null;
}

function associatedWallets(session) {
  const p = session?.player || {};
  return [...new Set([
    p.walletAddress,
    ...(p.associatedWalletAddresses || []),
    ...(p.linkedWalletAddresses || []),
    ...(p.walletAddresses || []),
  ].map(normalizeWallet).filter(Boolean))];
}

async function fetchInventory(wallet) {
  const url = new URL(`https://api.enterthegrotto.xyz/api/inventory/${wallet}`);
  url.searchParams.set('include_erc721', 'true');
  url.searchParams.set('limit', '500');
  const res = await fetch(url, { headers: { Accept: 'application/json' } });
  if (!res.ok) throw new Error(`Inventory ${res.status}`);
  return res.json();
}

function tokenMatches(item, gate) {
  const wantContract = gate.contractAddress.toLowerCase();
  const wantTokenId = String(gate.tokenId);
  return [item, item.asset, item.pack, item.game].filter(Boolean).some((x) => {
    const contract = String(x.contractAddress || x.collection_address || x.license_address || '').toLowerCase();
    const tokenId = String(x.tokenId || x.token_id || item.tokenId || item.token_id || '');
    return contract === wantContract && tokenId === wantTokenId;
  });
}

function inventoryHasToken(inv, gate) {
  return [inv.games, inv.gamePasses, inv.assetPacks, inv.assets, inv.nfts, inv.unregistered]
    .filter(Array.isArray)
    .some((items) => items.some((item) => tokenMatches(item, gate)));
}

async function refreshTokenGatedSkins(grotto, gameState) {
  const session = await grotto.getPlayer();
  const wallets = associatedWallets(session);
  const results = await Promise.allSettled(wallets.map(fetchInventory));
  const inventories = results.filter(r => r.status === 'fulfilled').map(r => r.value);
  const ownsGate = inventories.some((inv) => inventoryHasToken(inv, GATE));

  const set = new Set(gameState.unlockedSkins || []);
  if (ownsGate) set.add(SKIN_ID);
  else set.delete(SKIN_ID); // cosmetic revocation if token transferred
  gameState.unlockedSkins = [...set];

  if (ownsGate) {
    await grotto.event('skin_unlocked', { skinId: SKIN_ID, gate: 'token', contractAddress: GATE.contractAddress, tokenId: GATE.tokenId });
  }
  await grotto.save('default', gameState);
  return ownsGate;
}
```

Call after cloud save load:

```js
const grotto = await GrottoRuntime.ready({ timeoutMs: 10000 });
const save = await grotto.loadSave('default', DEFAULT_STATE);
gameState = save.state;
await refreshTokenGatedSkins(grotto, gameState);
startGame();
```

## Server-authoritative gates

Use server-side checks for prizes, ranked advantages, paid rewards, mints, tradeable items, or anything economically meaningful.

Flow:

1. Client sends its Grotto runtime token to your Railway/Supabase backend.
2. Backend verifies the runtime session with Grotto or an approved server endpoint.
3. Backend resolves associated wallets for that player.
4. Backend queries inventory/indexer from a trusted server environment.
5. Backend grants or denies the entitlement.
6. Client receives only `{ entitled: true/false }`.

Never ship indexer keys, Supabase service-role keys, mint keys, or admin secrets in browser files.

## Caching/revocation

- Cache cosmetic checks for 30-120 seconds.
- Recheck on boot, account change, and before valuable rewards.
- Cosmetics can disappear when ownership disappears.
- Rewards should store an entitlement record with grant reason and audit metadata.
