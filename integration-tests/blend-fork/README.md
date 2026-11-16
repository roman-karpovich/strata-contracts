# Blend fork integration test

Runs the current epoch manager and Blend adapter WASM against captured
**testnet** Blend state, using Kanatoko. This exercises the documented risk B2
without a deployment, credentials or live RPC calls during replay.

From the repository root, with the pinned Rust toolchain and Stellar CLI:

```sh
stellar contract build --locked --package strata-epoch-manager --out-dir integration-tests/blend-fork/wasm
stellar contract build --locked --package strata-blend-adapter --out-dir integration-tests/blend-fork/wasm
cargo test --locked --manifest-path integration-tests/blend-fork/Cargo.toml
```

The WASM is built from the current checkout, not a committed old version.
The separate test workspace uses a Protocol 30 Host to replay the fixture;
the production contract workspace keeps its existing SDK 27 dependency.

## What it checks

- An over-sized withdrawal is clamped to the supplied position.
- Calling `refresh()` every 90 days renews the pool position through a year.
- With the captured pool's 94.7% utilisation and no intervening pool activity,
  accrued liabilities exceed supply and `settle()` rejects with Blend error
  1207. The epoch stays open.
- After a local lender supplies liquidity, settlement can be retried; the
  senior target, junior remainder and both claims match the redeemed value.

This checks the existing documented behavior, not a fix for B2. It is a frozen
scenario, not a prediction of future pool conditions or production readiness.

## Fixture and limits

`fixtures/testnet-blend.json` was captured at testnet ledger 5648731,
Protocol 30, on 2026-11-09. It contains the pool, native token contract and
entries used by the scenario. Authorisation is mocked, accounts are funded
locally, and time/sequence are advanced at five seconds per ledger. State
archival is not modeled; the TTL assertion tests renewal before expiry.

The two Strata contracts need each other's addresses, so the test derives
both from a local deployer and salts before registration, as deployment does.
The two current WASM code keys are explicitly initialized as empty in the
local Host overlay before registration. This permits different compiler
output hashes without claiming that those keys were absent on testnet.
The committed capture and other network-key coverage are unchanged.
After registration, `replace_wasm` with the same bytes marks those executables
as local to Kanatoko while preserving both constructors' state.

CI builds current WASM and then replays this capture. After dependencies have
been fetched, use `cargo test --locked --offline --manifest-path
integration-tests/blend-fork/Cargo.toml`. A read outside the capture fails;
there is no fallback to live state. The ignored `recapture` test is an explicit
read-only testnet refresh, and can change the scenario's initial conditions.

Adapted from the independent [reproduction for #37](https://github.com/roman-karpovich/kanatoko-strata-check).
See `NOTICE` for fixture attribution.
