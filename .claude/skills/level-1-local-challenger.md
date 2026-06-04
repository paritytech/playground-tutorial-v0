---
quest: level-1
title: Local Challenger — AI context
---

# Context for Claude / AI pair

You are helping a developer complete **Level 1: Local Challenger**.

Read [00-overview.md](00-overview.md) first for the SDK package set and the
critical invariants. This file is the account/signing + local-game specifics.

## Starting state

- Host-managed product account via `@novasamatech/host-api-wrapper`
  (`createAccountsProvider(sandboxTransport).getProductAccount(identifier, 0)`)
- Best-of-3 vs computer
- Results saved in `localStorage` under key `rps-game:<h160Address>`
- Profile card showing W/L/D, win rate, last 10 games
- No smart contracts, no Bulletin, no Statement Store

## Goal

Ship a **modded** version to the developer's own `.dot` domain on Paseo Next v2.
Modding can be visual (theming, emoji sets) or behavioral (computer personality,
trash-talk, sound). Contract/chain changes are out of scope for this level.

## Runtime requirements

- **Polkadot Desktop ≥ 0.7.5** — accepts both `.dot` domains and `localhost:PORT`
  identifiers and ships the `host-api` 0.8.x wire protocol the SDK targets.
- **Polkadot mobile (v2)** if signing on a paired phone.

## Packages in play

Use the **latest** published versions (do not pin):

- `@novasamatech/host-api-wrapper` — accounts, signer, providers, preimage, permissions
- `@novasamatech/host-api` — host error types
- `@parity/product-sdk` — host detection
- `@parity/product-sdk-address` — `ss58ToH160`
- `polkadot-api`

The frozen `@novasamatech/product-sdk` (0.7.9-4) is no longer used — everything
the app imported from it now comes from `@novasamatech/host-api-wrapper`.

## Account flow (`src/utils.ts` pattern)

```ts
import {
  createAccountsProvider, sandboxTransport, type ProductAccount,
} from "@novasamatech/host-api-wrapper";
import { ss58ToH160 } from "@parity/product-sdk-address";
import { AccountId } from "polkadot-api";

// 0.8.x: createAccountsProvider takes the sandbox transport explicitly.
const accountsProvider = createAccountsProvider(sandboxTransport);

// Identifier = window.location.host VERBATIM. The host scopes signing to this
// exact context; appending `.dot` or extracting a label makes signing fail with
// `permission denied {expected: 'localhost:3000', got: 'localhost:3000.dot'}`.
const [identifier, derivationIndex] = getAppAccountId();   // [window.location.host, 0]
const result = await accountsProvider.getProductAccount(identifier, derivationIndex);
const { publicKey } = result.value;
const productAccount: ProductAccount = { dotNsIdentifier: identifier, derivationIndex, publicKey };

// "createTransaction" routes through the host's create-transaction RPC and
// bypasses PJS-signer, which throws on Paseo Next v2 signed extensions
// (AsPgas, AsRingAlias, CheckWeight, WeightReclaim).
const signer = accountsProvider.getProductAccountSigner(productAccount, "createTransaction");

const ss58 = AccountId().dec(publicKey);
const h160Address = ss58ToH160(ss58);   // pallet-revive / contract keying
```

## Common gotchas

- **Login only works inside Polkadot Desktop / dot.li.** No extension or plain
  browser fallback — the SDK is host-only by design.
- **`signerType: "createTransaction"` is mandatory.** The default `"signPayload"`
  goes through PJS-signer and throws on Paseo Next v2's custom signed extensions.
- **Identifier must be `window.location.host` verbatim** — for both `localhost`
  and `.dot`. Do NOT append `.dot` or extract a hostname label; that breaks
  signing. (Older notes said the opposite — they predate the host change.)
- **localStorage keys are per-account** (`rps-game:<address>`). Switching accounts
  swaps the profile.
- **Don't prompt to add contracts/Bulletin** — that's Level 2 / Level 3.

## Ship checklist

1. `npm run build:frontend` produces `dist/`
2. `dot deploy` (or `bulletin-deploy ./dist <name>.dot`) — Paseo Next v2 bulletin
3. Open `<name>.dot` inside Polkadot Desktop ≥ 0.7.5 to verify

## Do NOT

- Don't reintroduce `@novasamatech/product-sdk` — it's frozen; use `host-api-wrapper`.
- Don't append `.dot` to the identifier or extract a label — use the raw host.
- Don't use `@parity/product-sdk-signer`'s `SignerManager` — it calls
  `getLegacyAccounts()` which the new desktop/android hosts reject.
