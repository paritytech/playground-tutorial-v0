---
title: RPS architecture & SDK — START HERE
---

# Start here (AI context overview)

Rock Paper Scissors is a **Polkadot product-SDK app** that runs inside the
**Polkadot Desktop host** (and dot.li iframe). It has four tutorial levels, but
the production reference (`main`) wires them all together:

1. **Local game** — best-of-3 vs computer, account via the host.
2. **Bulletin storage** — game history JSON stored content-addressed; a CID pointer kept per account.
3. **Leaderboard contract** — a Rust/PVM contract on Paseo Asset Hub Next indexes players by address and stores `(cid, points)`.
4. **Multiplayer** — real-time commit-reveal over the Statement Store.

When starting fresh, read this file first, then the matching `level-N-*.md`.
The single most important source file is **`src/utils.ts`** — it owns the
account flow, signing, Bulletin upload, contract manager, and account mapping.
The contract is **`contracts/leaderboard/lib.rs`**.

## SDK packages (always use the latest published versions — do NOT pin)

Install with `npm install <pkg>@latest`. The old `@novasamatech/product-sdk`
(frozen at 0.7.9-4) is **gone** — development moved to `host-api-wrapper`.

| Package | What it provides |
|---|---|
| `@novasamatech/host-api-wrapper` | `createAccountsProvider(sandboxTransport)`, `getProductAccountSigner`, `createPapiProvider`, `preimageManager`, `requestPermission`, `sandboxTransport`, `ProductAccount` |
| `@novasamatech/host-api` | host error types (`RequestCredentialsErr`) |
| `@parity/product-sdk` | host detection (`isInsideContainerSync`) and helpers; subpath exports (`/host`, `/contracts`, …) |
| `@parity/product-sdk-contracts` | `ContractManager`, `createContractRuntimeFromClient`, `ensureContractAccountMapped` |
| `@parity/product-sdk-descriptors` | chain descriptor `paseo_asset_hub` |
| `@parity/product-sdk-address` | `ss58ToH160` (SS58 → H160 the way pallet-revive keys contracts) |
| `@parity/product-sdk-tx` | tx helpers used by the contracts layer |
| `@parity/product-sdk-statement-store` | `StatementStoreClient` (multiplayer) |
| `polkadot-api`, `@polkadot-api/ws-provider` | PAPI client + WS provider |
| `@noble/hashes`, `multiformats` | Bulletin CID calculation |

### Build gotcha — polkadot-api dep dedup

`host-api-wrapper` pulls both a modern `@polkadot-api/json-rpc-provider-proxy`
(needs `isResponse`) and, via the smoldot light-client path, an ancient
`@polkadot-api/json-rpc-provider@0.0.1` that lacks it. If the bundler errors
with `"isResponse" is not exported`, add an `overrides` entry forcing the
provider to the version the rest of the polkadot-api tree uses (currently
`0.2.0`):

```jsonc
"overrides": { "@polkadot-api/json-rpc-provider": "0.2.0" }
```

## Network — Paseo Next v2 (v1 retired 2026-05-20)

- **Asset Hub Next** (contracts): `wss://paseo-asset-hub-next-rpc.polkadot.io`
- **Bulletin Next**: `wss://paseo-bulletin-next-rpc.polkadot.io`
- **IPFS gateway**: `https://paseo-bulletin-next-ipfs.polkadot.io/ipfs/<cid>`
- **CDM registry** (flat-CDM): `0xf62c2ece29cd8df2e10040ecfa5a894a5c5d9cb0`
  — confirm against the current CDM release before a fresh deploy; it has changed
  across releases and is stored in `cdm.json` under the top-level `registry` key.

## Critical invariants (get these wrong → nothing signs or reads)

1. **Product identifier = `window.location.host` verbatim.** Do NOT append
   `.dot` or extract a label. The host scopes signing to the raw host context;
   any divergence fails with `permission denied {expected: 'localhost:3000',
   got: 'localhost:3000.dot'}`. The same identifier feeds `getProductAccount`,
   `productAccount.dotNsIdentifier`, and the statement-store `accountId`.
2. **Signer must be `getProductAccountSigner(productAccount, "createTransaction")`.**
   The default `"signPayload"` routes through PJS, which rejects Paseo Next v2's
   signed extensions (AsPgas, AsRingAlias, …).
3. **Every SS58 origin must be Revive-mapped before it touches a contract** —
   including read-only queries and the live-registry lookup. Unmapped → the
   dry-run fails with `Revive::AccountUnmapped`. Map with
   `ensureContractAccountMapped(runtime, address, signer)` (one-time signature).
4. **Contract ABI uses Solidity `address`** (not `bytes20`). Pass `0x…` H160 hex
   strings to the encoder; never hand-build `Binary`/`FixedSizeBinary`.
5. **Contract resolves live from the registry** (`ContractManager.fromLiveClient`),
   not from the cdm.json snapshot — so a redeploy is picked up without a new
   cdm.json. The lookup is a contract query, hence invariant #3.

## App is auth-gated

Every page renders only after a connected product account exists (see
`App.tsx`). There is no public, pre-sign-in contract read — so the connected
account is always available as the query/registry origin and no separate
read-only origin is needed.
