# Grotto Runtime SDK Contract

This is the intended public browser API exposed by `https://api.enterthegrotto.xyz/sdk/grotto-game-runtime.v1.js`.

## Global

```ts
declare global {
  interface Window {
    GrottoRuntime: GrottoRuntimeGlobal;
  }
}
```

## Types

```ts
type GrottoRuntimeConfig = {
  apiBaseUrl: string;
  gameId: string;
  sessionId: string;
  expiresAt?: string;
  expiresIn?: number;
  scopes: string[];
};

type GrottoPlayerSession = {
  sessionId?: string;
  gameId: string;
  authenticated: boolean;
  player: {
    id: string;
    walletAddress?: string | null;
    displayName?: string | null;
    avatar?: string | null;
  };
  scopes: string[];
  expiresAt?: string;
};

type GrottoSave<T = unknown> = {
  slot: string;
  version: number;
  updatedAt?: string;
  state: T;
};

type GrottoAutosaveOptions<T> = {
  slot?: string;
  defaultState: T;
  getState: () => T;
  applyState?: (state: T) => void;
  intervalMs?: number;
  onSaved?: (save: GrottoSave<T>) => void;
  onError?: (error: unknown) => void;
  onConflict?: (conflict: unknown) => void;
};

type GrottoAutosave = {
  start: () => Promise<void>;
  stop: () => void;
  markDirty: () => void;
  flush: () => Promise<boolean>;
};
```

## API

```ts
type GrottoRuntimeClient = {
  runtime: GrottoRuntimeConfig;
  getPlayer: () => Promise<GrottoPlayerSession>;
  loadSave: <T>(slot: string, defaultState: T) => Promise<GrottoSave<T>>;
  save: <T>(slot: string, state: T, options?: { baseVersion?: number }) => Promise<GrottoSave<T>>;
  deleteSave: (slot: string) => Promise<{ success: true }>;
  event: (type: string, payload?: Record<string, unknown>) => Promise<{ success: true }>;
  heartbeat: () => Promise<{ success: true; expiresAt?: string }>;
  getMultiplayerToken: (options?: { room?: string }) => Promise<unknown>;
  createAutosave: <T>(options: GrottoAutosaveOptions<T>) => GrottoAutosave;
};

type GrottoRuntimeGlobal = {
  ready: (options?: { timeoutMs?: number }) => Promise<GrottoRuntimeClient>;
};
```
