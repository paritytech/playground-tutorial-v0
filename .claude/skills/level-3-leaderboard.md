---
quest: level-3
title: The Leaderboard — AI context
---

# Context for Claude / AI pair

You are helping a developer complete **Level 3: The Leaderboard**.

Read [00-overview.md](00-overview.md) first — it covers the SDK package set,
the network, the registry, and the five critical invariants. This file is the
contract + deploy specifics.

## Goal

Deploy a Rust/PVM contract on **Paseo Asset Hub Next** that indexes players by
address. The contract stores `(cid, points)` per player; the game JSON itself
stays on Bulletin (Level 2). The contract is the index; Bulletin is the data.

## Toolchain

- `rustup`
- `cargo-pvm-contract` (the `pvm-contract-sdk` contract builder). Install the
  latest from the release; if it still documents an SDK branch, use the current
  one (`sm/cdm`), not any legacy `charles/cdm-integration` branch:
  ```sh
  HOST_TARGET=$(rustc -vV | awk '/^host:/ {print $2}')
  cargo install --force --locked --target "$HOST_TARGET" \
    --git https://github.com/paritytech/cargo-pvm-contract.git --branch sm/cdm \
    cargo-pvm-contract
  ```
  (Use the crates.io release once published.)
- `cdm` CLI (latest). The current `cdm` takes `--registry-address` as a
  first-class flag — there is **no** binary-patched `cdm-paseo-next` and **no**
  vendored `contract-registry` crate anymore. If you see those in old notes,
  ignore them; the flat-CDM registry already exists on v2.
- A funded, mapped account: `cdm account set` imports a mnemonic to
  `~/.cdm/accounts.json`; `cdm account bal -n paseo` checks PAS + Bulletin
  allowance; `cdm account map -n paseo` does the one-time Revive mapping
  (required before the first deploy). Fund via
  `https://faucet.polkadot.io/?parachain=1500`.

## Cargo manifests

**Workspace `Cargo.toml`** — one SDK dependency:

```toml
[workspace.dependencies]
pvm-contract-sdk = { git = "https://github.com/paritytech/cargo-pvm-contract", branch = "sm/cdm", features = ["alloc"] }
polkavm-derive = "0.31"
picoalloc = "5.2"
```

No `cdm`, `pvm_contract`, or `parity-scale-codec` unless the contract starts
importing other CDM packages (`cdm::import!`) or adds SCALE structs.

**`contracts/leaderboard/Cargo.toml`** — the package name lives in Cargo
metadata, NOT in the Rust macro:

```toml
[features]
abi-gen = ["pvm-contract-sdk/abi-gen"]

[package.metadata.cdm]
package = "@rps/leaderboard"

[dependencies]
pvm-contract-sdk = { workspace = true }
polkavm-derive = { workspace = true }
picoalloc = { workspace = true }
```

Package names are globally owned in the registry — use an app-specific
namespace (`@rps/leaderboard`), not `@example/...`.

## Contract shape — receiver-based SDK (`contracts/leaderboard/lib.rs`)

The macro is `#[pvm_contract_sdk::contract(...)]` on a module with a storage
`struct` (slots via `#[slot(N)]`), an `impl` with `#[constructor]` / `#[method]`
receivers (`&self` / `&mut self`), and typed errors via `sol_revert_enum!`.
Return `Address` (encodes as Solidity `address`), not `[u8; 20]`/`bytes20`.

```rust
#![cfg_attr(not(feature = "abi-gen"), no_main, no_std)]

#[pvm_contract_sdk::contract(allocator = "pico", allocator_size = 4096)]
mod leaderboard {
    use alloc::string::String;
    use pvm_contract_sdk::{Address, HostApi, Lazy, Mapping};

    pvm_contract_sdk::sol_revert_enum! {
        pub enum Error {
            AlreadyRegistered(AlreadyRegistered),
            NotRegistered(NotRegistered),
            IndexOutOfBounds(IndexOutOfBounds),
        }
    }
    #[derive(Debug, pvm_contract_sdk::SolError)] pub struct AlreadyRegistered;
    #[derive(Debug, pvm_contract_sdk::SolError)] pub struct NotRegistered;
    #[derive(Debug, pvm_contract_sdk::SolError)] pub struct IndexOutOfBounds;

    pub struct Leaderboard {
        #[slot(0)] player_count: Lazy<u64>,
        #[slot(1)] player_at: Mapping<u64, [u8; 20]>,
        #[slot(2)] is_registered: Mapping<[u8; 20], bool>,
        #[slot(3)] player_cid: Mapping<[u8; 20], String>,
        #[slot(4)] player_points: Mapping<[u8; 20], i64>,
    }

    impl Leaderboard {
        #[pvm_contract_sdk::constructor]
        pub fn new(&mut self) { self.player_count.set(&0); }

        #[pvm_contract_sdk::method]
        pub fn register(&mut self) -> Result<u64, Error> {
            let caller = self.caller();
            if self.is_registered.get(&caller.0) { return Err(AlreadyRegistered.into()); }
            let idx = self.player_count.get();
            self.player_at.insert(&idx, &caller.0);
            self.is_registered.insert(&caller.0, &true);
            self.player_points.insert(&caller.0, &0);
            self.player_count.set(&(idx + 1));
            Ok(idx)
        }

        #[pvm_contract_sdk::method]
        pub fn update_result(&mut self, new_cid: String, points_delta: i64) -> Result<(), Error> {
            let caller = self.caller();
            if !self.is_registered.get(&caller.0) { return Err(NotRegistered.into()); }
            self.player_cid.insert(&caller.0, &new_cid);
            let current = self.player_points.get(&caller.0);
            self.player_points.insert(&caller.0, &(current + points_delta));
            Ok(())
        }

        #[pvm_contract_sdk::method]
        pub fn get_player_at(&self, index: u64) -> Result<Address, Error> {
            if index >= self.player_count.get() { return Err(IndexOutOfBounds.into()); }
            Ok(Address(self.player_at.get(&index)))
        }

        // get_player_count() -> u64; get_player_cid(Address) -> String;
        // get_player_points(Address) -> i64; is_registered(Address) -> bool
        // ... same receiver pattern ...

        fn caller(&self) -> Address {
            let mut buf = [0u8; 20];
            self.host().caller(&mut buf);
            Address(buf)
        }
    }
}
```

`cargo pvm-contract build` writes `target/release/leaderboard.polkavm` and
`leaderboard.abi.json` — confirm the ABI shows `getPlayerAt() -> address`,
`getPlayerCid(address)`, etc., and `register` as `nonpayable`.

## Build + deploy flow (from scratch)

```sh
# 1. compile the contract
cargo pvm-contract build --manifest-path Cargo.toml -p leaderboard

# 2. regenerate the CDM manifest from the new package metadata
rm -rf .cdm cdm.json
cdm build

# 3. map the signing account on Revive (idempotent; needed before first deploy)
cdm account map -n paseo

# 4. deploy + register into the flat-CDM registry
npm run deploy        # = cdm deploy -n paseo --registry-address 0xf62c2ece29cd8df2e10040ecfa5a894a5c5d9cb0 \
                      #     --assethub-url wss://paseo-asset-hub-next-rpc.polkadot.io \
                      #     --bulletin-url  wss://paseo-bulletin-next-rpc.polkadot.io

# 5. read the deployed address/abi back from the registry into cdm.json
cdm i -n paseo --registry-address 0xf62c2ece29cd8df2e10040ecfa5a894a5c5d9cb0 \
  --assethub-url wss://paseo-asset-hub-next-rpc.polkadot.io \
  --ipfs-gateway-url https://paseo-bulletin-next-ipfs.polkadot.io/ipfs @rps/leaderboard
```

The resulting `cdm.json` is the **flat** shape — top-level `registry`,
`dependencies: { "@rps/leaderboard": "latest" }`, and `contracts["@rps/leaderboard"]`
with `version`, `address`, `abi`, `metadataCid`. There is **no** `targets` block
and **no** target-hash key (`acc2c3b5…`) — those were the old shape.

## Frontend integration

`src/utils.ts` lazy-inits the contract on first method call (holding a chain
follow open at startup starves Bulletin preimage submits). The init **must map
the account before resolving live addresses** — `fromLiveClient` immediately
queries the registry, and an unmapped origin fails with `AccountUnmapped`:

```ts
import {
  ContractManager, createContractRuntimeFromClient, ensureContractAccountMapped,
} from "@parity/product-sdk-contracts";
import { paseo_asset_hub } from "@parity/product-sdk-descriptors/paseo-asset-hub";

await client.getChainSpecData();
await client.getBestBlocks();           // wake the chain follow first

// 1. map the connected product account (plain runtime, no registry query)
const initRuntime = createContractRuntimeFromClient(client, paseo_asset_hub);
await ensureContractAccountMapped(initRuntime, account.address, account.signer);

// 2. NOW resolve live addresses from the registry
const manager = await ContractManager.fromLiveClient(
  cdmJson, client, paseo_asset_hub,
  {
    defaultOrigin:  account.address,
    defaultSigner:  account.signer,
    registryOrigin: account.address,        // also a query origin → must be mapped
    libraries: ["@rps/leaderboard"],
  },
);
const lb = manager.getContract("@rps/leaderboard");

// queries return { success, value, gasRequired }; pass 0x… hex for address args
const res = await lb.getPlayerCid.query(asAddress(account));
await lb.register.tx();                                   // returns Result<u64, Error>
await lb.updateResult.tx(newCid, BigInt(pointsDelta));    // returns Result<(), Error>
```

`asAddress(hexOrAccount)` returns a `0x…` H160 string (the encoder accepts it
for the `address` type). Don't build `Binary`/`FixedSizeBinary` — cross-realm
class identity breaks when substrate-bindings hoists multiple copies.
`asBytes20` is kept as a deprecated alias.

## Common gotchas

- **`ContractLiveAddressResolutionError: Failed to resolve live address`** = the
  registry view query failed, almost always because the query origin isn't
  Revive-mapped. Map the account *before* `fromLiveClient` (see above). To
  confirm it's mapping (not a missing registration), a mapped origin returns
  `{ success: true, value: { isSome: true, value: "0x…" } }` while an unmapped
  one returns `success: false → Revive::AccountUnmapped`.
- **`permission denied {expected: 'localhost:3000', got: '...dot'}`** = the
  product identifier diverged from the host context. Use `window.location.host`
  verbatim (invariant #1 in the overview).
- **`PJS does not support this signed-extension`** = signer built without
  `"createTransaction"`.
- **`DuplicateContract`** on re-deploy is harmless (deterministic address).
- **`Invalid::Payment`** = the deployer needs PAS (Asset Hub for the contract,
  Bulletin allowance for the metadata upload).
- **H160 vs SS58**: the contract keys by H160 (`account.h160Address`); SS58
  (`account.address`) is only for origin/signing. Use `ss58ToH160` — don't roll
  your own keccak.
- **Points are `i64`** (losses subtract). **`register()` once per player** before
  any `update_result()` — check `isRegistered()` and auto-register on first save.

## Acceptance check

- `cargo pvm-contract build` + `cdm build` + `npm run deploy` + `cdm i` succeed;
  `cdm.json` has the flat shape with the registry and `@rps/leaderboard` address.
- A fresh player triggers one `Revive.map_account()` signature, then `register()`,
  then `update_result()` per match.
- Page refresh resolves the current address live from the registry.
- Leaderboard lists registered players sorted by points.

## Do NOT

- Don't use the old `#[pvm::storage]` / `#[pvm::contract(cdm = "...")]` macro —
  that's the legacy SDK. Use `#[pvm_contract_sdk::contract]` with receivers.
- Don't return `bytes20`; return `Address`.
- Don't use `ContractManager.fromClient` (snapshot) unless you intentionally want
  the installed address — the app wants live resolution.
- Don't import `@polkadot-api/sdk-ink` — dropped from product-sdk-contracts.
- Don't reintroduce the cdm 0.1.0 binary patch or vendored `contract-registry`
  crate — the flat-CDM registry exists and `--registry-address` is supported.
- Don't construct contract addresses by hand; deploy writes them into `cdm.json`.
