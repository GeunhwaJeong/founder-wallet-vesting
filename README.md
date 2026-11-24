# founder-wallet-vesting

Vesting contracts that lock the Haneul founder allocation on-chain, together with the full deployment record so anyone can verify the lockup independently.

The Move packages in this repository are a port of [OpenZeppelin Contracts for Sui](https://github.com/OpenZeppelin/contracts-sui) (`math/core` and `contracts/finance`, commit `57399c7`). The only source change is the framework namespace (`sui::` renamed to `haneul::`) plus package metadata; the vesting logic, math, and tests are unmodified. All 887 bundled unit tests (finance 104, math 783) pass against the Haneul framework.

Both packages are **immutable**: their `UpgradeCap`s were burned in the same session as the deployment, so this code can never be modified or replaced by anyone, including the publisher.

## Deployments

Network: **Haneul Mainnet**, chain id `a0053d9e`, public gRPC `158.69.54.239:9000`

| Package | Address | Publish tx |
|---|---|---|
| `openzeppelin_math` | `0x43214488f62ab77c071b54e61e28bea721c0d9ebd69bb53f810ca911cac1f23a` | `85gf2YDapCicB7nHtdhv3KVXGzpbGf257TGDtePpWJRq` |
| `openzeppelin_finance` | `0x3167daad32500f2293efbadf6bb8997da35e5ac309aeefce6644e125b48920f3` | `H192dQmHpweXmSvYB95JmLccouRgqZUN6oSCBDaKjAZ1` |

UpgradeCap burn (both packages, `0x2::package::make_immutable`): `5R55GnSkPNnZcSfY1Ypoq5nh9qvVpSHTY4Paciv8Pb4g`

Both package objects report `owner: Immutable` on-chain.

## Founder lockup

| Field | Value |
|---|---|
| VestingWallet (shared object) | `0x461c68295d990769c4c602dadceaa2a16a7543b4535e93e16de211c316ad1d17` |
| Locked amount | 1,990,000,000 HANEUL (19.9% of total supply) |
| Beneficiary | `0x804915e146733616462d0aacce820589e2ebfb5f7e4d9af15cce2f1a5ebcb031` |
| Vesting start | 2029-03-30 00:00:00 KST (`start_ms = 1869490800000`) |
| Vesting end | 2032-03-30 00:00:00 KST (linear per millisecond, 1,096 days) |
| Cliff | none (`cliff_ms = 0`; the curve returns zero before start) |
| Lock tx | `EaAd2ENkY6bV8EUFhUrTxDmjEEpnsVicXg89hd3zo1qp` |

Properties enforced by the deployed bytecode:

- **Nothing is claimable before 2029-03-30.** The curve evaluates to zero before the start timestamp.
- **Release is permissionless, payout is not.** The wallet is a shared object, so any address can execute `release`. The contract pays only to the beneficiary recorded at creation. `release` takes no `TxContext`, so it cannot even observe who the caller is.
- **The beneficiary cannot change.** The module exposes no setter for the beneficiary field.
- **No early-withdrawal path exists.** The only functions that move funds out are the schedule-gated `release` and, after the schedule ends and the balance is zero, a teardown of the empty wallet.
- **The code is frozen.** With both `UpgradeCap`s burned, none of the above can ever be amended.

## Verify it yourself

The commands below use [grpcurl](https://github.com/fullstorydev/grpcurl).

Read the lockup wallet state (balance, released amount, beneficiary, schedule):

```bash
grpcurl -plaintext -d '{
  "object_id": "0x461c68295d990769c4c602dadceaa2a16a7543b4535e93e16de211c316ad1d17",
  "read_mask": {"paths": ["owner", "json"]}
}' 158.69.54.239:9000 haneul.rpc.v2.LedgerService/GetObject
```

Confirm the packages are immutable:

```bash
grpcurl -plaintext -d '{
  "object_id": "0x3167daad32500f2293efbadf6bb8997da35e5ac309aeefce6644e125b48920f3",
  "read_mask": {"paths": ["owner"]}
}' 158.69.54.239:9000 haneul.rpc.v2.LedgerService/GetObject
```

Track cumulative payouts to the beneficiary (the `addressBalance` field):

```bash
grpcurl -plaintext -d '{
  "owner": "0x804915e146733616462d0aacce820589e2ebfb5f7e4d9af15cce2f1a5ebcb031",
  "coin_type": "0x2::haneul::HANEUL"
}' 158.69.54.239:9000 haneul.rpc.v2.StateService/GetBalance
```

## Mainnet rehearsal

Before the lockup, the exact release path was exercised on mainnet with a 10-minute schedule (wallet `0xb8400fdbed75fb0746917c92b73c890010a7ce5f784cd895bd7af6f1c8538627`, funded with 1 HANEUL). All three release transactions were signed by a third-party address, not the beneficiary:

| Phase | Tx | Result |
|---|---|---|
| Before start | `31Kv8Mj3mKzWmvvja4itYtYBrhXuiyb4sfyy4WU5jNjh` | paid 0 |
| Mid-schedule (50%) | `7NoGahmDuqRsmwjUqoRy7TwKFoJYRKnvPtwDY2y7ox8N` | paid 0.503806666 HANEUL |
| After end | `3K2uUepGDnL7k3gTqf8CETcSuJvXs4vA1pnzLcMVzbCe` | paid remainder, total exactly 1.0 |

Every payout landed in the beneficiary address balance; the callers received nothing.

## Repository layout

```
math/       openzeppelin_math: overflow-safe integer arithmetic (mul_div in wide
            intermediates, rounding modes). Dependency of finance.
finance/    openzeppelin_finance: the vesting_wallet primitive and the
            vesting_wallet_linear schedule used by the lockup.
```

Each package records its published addresses in `Published.toml`. Run the test suites with `haneul move test` inside each package directory.

## License

MIT, inherited from OpenZeppelin Contracts for Sui. See [LICENSE](LICENSE).
