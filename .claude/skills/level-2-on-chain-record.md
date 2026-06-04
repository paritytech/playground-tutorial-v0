---
quest: level-2
title: On-Chain Record — AI context
---

# Context for Claude / AI pair

You are helping a developer complete **Level 2: On-Chain Record**.

Read [00-overview.md](00-overview.md) first for the SDK package set and invariants.

## Goal

Move the game history JSON itself to Bulletin Chain (content-addressed). Keep a **CID pointer in localStorage** per account (`rps-game-cid:<address>`) so the app can resolve the latest history back to the player on reload. In Level 3, this pointer moves out of localStorage and into the leaderboard smart contract.

## Network

Paseo Next v2 (v1 retired 2026-05-20):

- **Asset Hub Next**: `wss://paseo-asset-hub-next-rpc.polkadot.io` (genesis `0x173cea9d…`)
- **Bulletin Next**: `wss://paseo-bulletin-next-rpc.polkadot.io`
- **IPFS gateway**: `https://paseo-bulletin-next-ipfs.polkadot.io/ipfs/<cid>` (primary; fall back to `dweb.link`, `ipfs.io`, `nftstorage.link`)

## What to use (current SDK)

The repo does NOT use `@parity/product-sdk-bulletin` directly. It uses the **host's preimage manager** because Polkadot Desktop has an optimized path: upload bytes via `preimageManager.submit(bytes)`, host turns them into a Bulletin extrinsic + IPFS pin. This works in dev mode (where opening an Asset Hub chain client is awkward) and avoids spinning up a second chain client just for Bulletin.

```ts
import { preimageManager, requestPermission } from "@novasamatech/host-api-wrapper";
import { blake2b } from "@noble/hashes/blake2.js";
import { CID } from "multiformats/cid";
import * as raw from "multiformats/codecs/raw";

// 1) Pre-compute the CID locally so the caller can use it before the tx lands.
//    Bulletin uses blake2b-256 multihash (code 0xb220) wrapped in a CIDv1 raw codec.
function calculateCID(bytes: Uint8Array): string {
    const hash = blake2b(bytes, { dkLen: 32 });
    const multihash = /* varint(0xb220) | varint(32) | hash */;
    return CID.createV1(raw.code, { code: 0xb220, size: 32, bytes: multihash, digest: hash }).toString();
}

// 2) Ask the host once for PreimageSubmit permission, then submit.
async function uploadToBulletin(bytes: Uint8Array): Promise<string> {
    await requestPermission({ tag: "PreimageSubmit", value: undefined });
    const cid = calculateCID(bytes);
    await preimageManager.submit(bytes);
    return cid;
}
```

The repo already exports `uploadToBulletin(account, bytes)` and `calculateCID(bytes)` in `src/utils.ts` — use those instead of rebuilding.

## CID pointer flow (the Level 2 addition)

```ts
const CID_KEY = (addr: string) => `rps-game-cid:${addr}`;

// On match end:
const playerData = { games: [...existing, newGame], wins, losses, draws };
const bytes = new TextEncoder().encode(JSON.stringify(playerData));
const cid = await uploadToBulletin(account, bytes);
localStorage.setItem(CID_KEY(account.h160Address), cid);

// On profile load:
const cid = localStorage.getItem(CID_KEY(account.h160Address));
if (cid) {
    const bytes = await fetchFromGateway(cid);
    const data = JSON.parse(new TextDecoder().decode(bytes));
    // render profile from `data`
}
```

`fetchFromGateway(cid)` in `utils.ts` races all four gateways with `Promise.any` — first responder wins. Use it instead of hardcoding a single gateway URL.

## Common gotchas

- **PreimageSubmit permission is per-session.** First call shows the host prompt; subsequent calls within the session reuse the grant. The repo caches granted permissions in a `Set<string>` (`_grantedPermissions`).
- **The host owns the Bulletin connection.** You do NOT need to open `wss://paseo-bulletin-next-rpc.polkadot.io` from the SPA — `preimageManager.submit` proxies through the host's open chain client.
- **No separate Bulletin faucet is needed** when using `preimageManager.submit` — the host's account pays storage. (Standalone `bulletin-deploy` CLI is a separate codepath that uses your own funded account; that's only for `dot deploy`, not for in-app uploads.)
- **CIDs are deterministic.** Re-uploading identical bytes returns the same CID — useful for idempotency, but don't rely on it as a "did this upload succeed?" check; query the gateway instead.
- **Payload size cap is around 1 MB** per preimage. History grows with each game but typically stays small; no need to paginate at this level.

## Acceptance check

- Log the CID, paste it into `https://paseo-bulletin-next-ipfs.polkadot.io/ipfs/<cid>`, confirm JSON is readable
- Refresh the page → profile reloads from Bulletin via the localStorage CID pointer

## Do NOT

- Don't add a smart contract yet — Level 3
- Don't add multiplayer — Level 4
- Don't import `@parity/product-sdk-bulletin` and call `BulletinClient.create()` yourself — the host's `preimageManager` is the right path for in-app uploads; `BulletinClient` is for standalone Node.js / CLI tooling
- Don't hardcode old paseo endpoints (`asset-hub-paseo-rpc.n.dwellir.com`, `paseo-ipfs.polkadot.io`, etc.) — Paseo v1 is retired
