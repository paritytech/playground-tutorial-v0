---
quest: level-4
title: Multiplayer — AI context
---

# Context for Claude / AI pair

You are helping a developer complete **Level 4: Multiplayer**.

Read [00-overview.md](00-overview.md) first for the SDK package set and invariants.

## Goal

Two accounts play a real-time best-of-3 over Statement Store using commit-reveal anti-cheat. Results save to Bulletin + leaderboard contract for both players (reuse Level 2 + 3 flows).

## Dependency

`@parity/product-sdk-statement-store` (latest — do not pin). Host mode required —
same host the Level 1 signer goes through. Builds on `@novasamatech/host-api-wrapper`
+ Polkadot Desktop ≥ 0.7.5. The `StatementStoreClient` API (constructor config,
`connect`/`publish`/`subscribe`/`destroy`, `ReceivedStatement.data`) is stable
across recent versions, so a fresh install on latest needs no code change.

## Connection pattern (current SDK)

```ts
import { StatementStoreClient } from "@parity/product-sdk-statement-store";

const client = new StatementStoreClient({
    appName: "rps-game",
    defaultTtlSeconds: 600,
});

await client.connect({
    mode: "host",
    accountId: account.productAccountId,    // [identifier, derivationIndex] tuple — see utils.ts
});
```

`account.productAccountId` is `[window.location.host, derivationIndex]`, set up in
`utils.ts` from `getProductIdentifier()`. Don't pass a literal like
`["rps-game.dot", 0]` — the identifier is the raw host and must match what the
host signed off on at account creation (same invariant as the Level 1 signer).

## Publish / subscribe

```ts
// Subscribe — receives EVERYONE's statements (including your own publishes).
client.subscribe<JoinMessage>((statement) => {
    const msg = statement.data;
    if (msg.peerId === account.h160Address) return;   // skip self
    // ... handle opponent message
});

// Publish — typed by `<T>`; client SCALE-encodes + signs + submits via host.
await client.publish<JoinMessage>(
    { type: "join", peerId: account.h160Address, ts: Date.now() },
    { topic2: roomCode },   // scope to room
);

// Teardown on unmount / game end
client.destroy();
```

## Channels (all scoped via `topic2: roomCode`)

```text
{roomCode}/presence/{peerId}        — join announcement
{roomCode}/commit/{round}/{peerId}  — SHA-256 hash of (move + salt)
{roomCode}/reveal/{round}/{peerId}  — move + salt revealed after both commits
```

Topic1 (app-level) is derived from `appName: "rps-game"` automatically — you don't set it manually.

## Commit-reveal protocol per round

1. Player picks move → generate `salt = crypto.randomUUID()`
2. Compute `hash = SHA256(move + salt)`
3. Publish commit
4. Wait for opponent's commit
5. Once both commits received, publish reveal (move + salt)
6. Verify opponent's reveal: `SHA256(reveal.move + reveal.salt) === storedCommit`
7. Determine round winner from the two moves

## Common gotchas

- **Host mode required** — public WebSocket endpoints to Bulletin do **not** expose `statement_*` RPC methods. The client routes through the host's native binary protocol (`createStatementStore()` under the hood). Must run inside Polkadot Desktop.
- **StatementSubmit permission is per-session** — first `publish()` triggers the host prompt. The repo's `ensurePermission("StatementSubmit")` helper caches the grant. If you instantiate the client inside a component, the first publish may show a permission modal — design the UX around that.
- **Stale closures are the #1 multiplayer bug.** `handleMessage` runs inside a subscribe callback that captures state at subscription time. Use `useRef` for `myMove`, `mySalt`, `opponentCommit`, `round`, `phase`. Update refs synchronously alongside `setState`. The repo's `MultiplayerGame.tsx` shows the right pattern.
- **Skip own messages.** Subscribe receives everyone's statements including your own. Filter on `statement.data.peerId === account.h160Address`.
- **Deduplication.** Subscribe may replay a statement during reconnect or poll. v0.2.3 has internal `seen` dedup but you can also dedup by `{type, round, peerId, timestamp}` for safety.
- **Phase machine:** `connecting → pick → waiting-commit → waiting-reveal → round-result → (pick | game-over)`. Transitions must be atomic; never allow two reveals for the same round.
- **Hash mismatch = abort.** If opponent's SHA256 doesn't match their earlier commit, don't process the round — treat as cheat attempt.
- **After match ends, save once.** Reuse the Level 2 `uploadToBulletin` + Level 3 `lb.updateResult.tx(...)` flow. Both players save independently to their own Bulletin CID + contract entry. Both must already have called `ensureContractAccountMapped` (Level 3) — if a player is first-time, expect a one-time `Revive.map_account()` signature prompt before the leaderboard tx.
- **`createTransaction` signerType propagates here too.** The Statement Store client uses `account.signer` under the hood for the SCALE-signed statements — that signer must be the one built with `signerType: "createTransaction"` (see [level-1](level-1-local-challenger.md)). If it was built with the default `"signPayload"`, the host will reject the statement-submit tx with `PJS does not support this signed-extension: AsPgas` on Paseo Next v2.

## Channel store helper (if used)

For per-room key-value state (e.g., scoreboard) the repo can use `ChannelStore`:

```ts
import { StatementStoreClient, ChannelStore } from "@parity/product-sdk-statement-store";

const channels = new ChannelStore<ScoreboardEntry>(client, { topic2: roomCode });
await channels.write("score", { p1: 2, p2: 1 });
channels.subscribe("score", (value) => { /* re-render */ });
```

Skip if you don't need it — the raw `publish` / `subscribe` flow above is enough for commit-reveal.

## Acceptance check

- Two accounts in two Polkadot Desktop windows can complete a full best-of-3
- Watch the console: neither side should see the opponent's move emoji before their own commit is published
- Intentionally break the reveal (edit `move` in devtools before publish) → other side detects hash mismatch
- Both leaderboards update with correct +/- points after the match
- New player on either side: see one `Revive.map_account()` prompt followed by the contract `register()` + `update_result()` txs

## Do NOT

- Don't use `@novasamatech/sdk-statement` directly — `@parity/product-sdk-statement-store` wraps it and gives you typed publish/subscribe + host-transport routing. Going lower-level loses the type safety and the dedup.
- Don't connect with `mode: "local"` — local mode hits the public WS endpoint which doesn't serve `statement_*` methods. Always `mode: "host"`.
- Don't hardcode room codes — use a 6-char random generator with a confusable-free alphabet (`ABCDEFGHJKLMNPQRSTUVWXYZ23456789`). The repo exports `generateRoomCode()` in `utils.ts`.
- Don't share one `StatementStoreClient` instance across rooms — `destroy()` it when the room ends and create a new one for the next match. Subscriptions are scoped to the client.
