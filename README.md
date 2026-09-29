# Refract Contracts

> Soroban smart contracts for [Refract](https://github.com/refract-protocol) — trustless, oracle-triggered parametric insurance on Stellar.

This repository holds the three on-chain contracts that make up Refract's settlement layer. For the protocol overview, backend, and web app see the sibling repositories:

| Repo | Role |
|---|---|
| **refract-contracts** (this repo) | Soroban contracts: pool, policy registry, oracle |
| `refract-backend` | Oracle monitoring, claim processing, REST/WebSocket API |
| `refract-frontend` | Next.js web app for policyholders & capital providers |

## Contracts

| Crate | Contract | Responsibility |
|---|---|---|
| `pool/` | `RefractPool` | Holds USDC risk capital, prices & sells policies, settles claims against oracle data |
| `policy/` | `RefractPolicyRegistry` | Queryable on-chain index of policies per holder (sidecar to the pool) |
| `oracle/` | `RefractOracle` | Permissioned price/event feed with staleness enforcement and trigger evaluation |

All amounts use **1e7 fixed-point** (the Stellar USDC convention). The contracts are `#![no_std]` and never touch floating point.

## Prerequisites

- [Rust](https://rustup.rs/) (stable) with the `wasm32-unknown-unknown` target:
  ```bash
  rustup target add wasm32-unknown-unknown
  ```
- [Stellar CLI](https://developers.stellar.org/docs/tools/developer-tools) for deployment (`stellar`)

## Build & test

```bash
cargo test                                          # run the unit-test suite
cargo fmt --all --check                             # formatting
cargo clippy --all-targets -- -D warnings           # lints
cargo build --target wasm32-unknown-unknown --release   # optimized wasm
```

The release `.wasm` artifacts land in `target/wasm32-unknown-unknown/release/`.

## Deploy (testnet)

Deploy in dependency order — the oracle first, then the pool, then the registry:

```bash
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/refract_oracle.wasm \
  --source alice --network testnet

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/refract_pool.wasm \
  --source alice --network testnet

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/refract_policy.wasm \
  --source alice --network testnet
```

Then `initialize` each contract (admin, USDC token address, and pool↔registry wiring).

## Authorization model

- **RefractPool** — `provide_capital`, `withdraw_capital`, and `buy_policy` require the caller's auth. `update_oracle` is admin-only.
- **RefractPool guardian (fast-path pause)** — the pool stores a `Guardian` address, distinct from the admin/timelock. The guardian's *only* power is to trigger the emergency pause instantly: `set_paused(true)` accepts either the admin/timelock **or** the guardian, with no timelock delay. Unpausing (`set_paused(false)`) is **admin/timelock-only** — a guardian can freeze the protocol fast but cannot unilaterally resume it, so a compromised guardian key is never worse than having none. The guardian role itself is settable/removable **only** through the full governance/timelock (admin) path; a guardian can never grant itself the role or extend its own term. Guardian powers do not extend to any other admin action (e.g. the oracle circuit-breaker clear stays admin/timelock-gated).
- **RefractOracle** — only registered relayers (or the admin) may `submit`; readings older than 30 minutes are rejected.
- **RefractPolicyRegistry** — only the registered pool contract or the admin may `register_policy` / `deactivate_policy`.

## Status

⚠️ **Pre-audit.** These contracts are testnet-only and have not had a professional security review. Do not deploy on mainnet. See [`SECURITY.md`](./SECURITY.md).

## Contributing

We welcome contributors — see [`CONTRIBUTING.md`](./CONTRIBUTING.md) and our [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

## License

[MIT](./LICENSE)
