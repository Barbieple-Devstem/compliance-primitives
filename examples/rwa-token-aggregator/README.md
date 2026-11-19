# rwa-token-aggregator

Reference RWA token whose `transfer` path calls **only**
[`compliance-aggregator`](../../contracts/compliance-aggregator)`.check_address`.
The aggregator is the single top-level gate and composes the primitives
itself:

```text
RwaAggregatorToken.transfer(from, to, amount)
  └── compliance-aggregator.check_address(party, allowed_codes)   (× from, to)
        ├── circuit-breaker.is_frozen()      — emergency stop, short-circuits
        ├── denylist-gate.check(party)
        └── jurisdiction-flag.is_permitted_jurisdiction(party, allowed_codes)
```

| Failed check (from the aggregator's breakdown) | Token error |
| --- | --- |
| `CircuitBreaker` | `Error::CircuitBreakerFrozen` |
| `Denylist` | `Error::DeniedByGate` |
| `Jurisdiction` | `Error::JurisdictionNotPermitted` |
| Aggregator call itself failed (e.g. `NoChecksRegistered`) | `Error::ComplianceUnavailable` (fail closed) |

## Why route through the aggregator?

Compare with [`rwa-token`](../rwa-token), which stores every primitive's
address and calls each one directly. Here the token stores one address. The
aggregator admin (ideally a [`multisig-admin`](../../contracts/multisig-admin))
can register, swap or clear the denylist, jurisdiction and circuit-breaker
contracts without changing or redeploying the token. See
`test_swapping_primitive_in_aggregator_needs_no_token_change`.

For the cost tradeoff between calling `check_address` per party and calling
`check_all` once, see "Choosing `check_address` vs. `check_all`" in the
aggregator's module docs.

## Pattern

The crate does not link the aggregator's `#[contractimpl]` exports. It uses a
`#[contractclient]` trait and mirrors `CheckKind` / `CheckResult` with the
same `#[contracttype]` shape. The primitive crates are `dev-dependencies` used
only by the tests.

## Build & test

```sh
cargo test -p rwa-token-aggregator
cargo build -p rwa-token-aggregator --target wasm32v1-none --release
```
