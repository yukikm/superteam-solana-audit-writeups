# Solana program-examples escrow (native): fix `take_offer` panic + rent-refund leak

Repo: https://github.com/solana-developers/program-examples

Component:
- `tokens/escrow/native/program`
- instruction handler: `TakeOffer::process`

Upstream PR: https://github.com/solana-developers/program-examples/pull/533

Fix (fork commit): https://github.com/yukikm/program-examples/commit/0e0cf8b

Patch (GitHub PR .patch): https://github.com/solana-developers/program-examples/pull/533.patch

## Summary
While auditing the **native escrow** example, I found two issues in `take_offer`:

1) **Incorrect post-transfer invariant check** that can **panic / deterministically fail** even when token transfers are correct.
2) **Rent-refund destination bug**: lamports from closing the offer account could be refunded to an arbitrary account provided by the taker (a value leak from the maker).

The fix makes both paths safe and aligns the example with best practices (example code is frequently copy-pasted into production).

## Issue 1 — Incorrect invariant (panic / deterministic failure)
### Root cause
In `tokens/escrow/native/program/src/instructions/take_offer.rs`, the code checked the maker’s Token-B post-transfer balance using the **wrong baseline variable**:

```rust
assert_eq!(
    maker_amount_b,
    taker_amount_a_before_transfer + offer.token_b_wanted_amount
);
```

`taker_amount_a_before_transfer` is the taker’s Token-A balance before receiving from the vault; it has nothing to do with the maker’s Token-B account.

### Impact
If the taker already has any non-zero Token-A balance (a normal scenario), `take_offer` can panic (via `assert_eq!`) and fail the transaction, even though transfers executed correctly. On-chain panics waste compute and are a reliability footgun for integrators.

### Fix
Track the correct baseline (`maker_amount_b_before_transfer`) and assert that the maker’s Token-B increased by exactly `offer.token_b_wanted_amount`.

## Issue 2 — Offer rent refund could be captured by taker (value leak)
### Root cause
When closing the offer account, the pre-fix code refunded its lamports to a `payer` account supplied in the instruction accounts. The taker can set `payer = taker`, allowing the taker to capture the maker-funded rent.

### Impact
**Value leakage** from maker → taker via "rent rebate theft". Even if small per offer, this is a clear economic bug and a bad pattern to teach in an example repository.

### Fix
Refund offer account lamports to the **maker** instead of an arbitrary `payer`.

## Reproduction
### Issue 1 (panic)
1. Ensure the taker already holds a non-zero Token-A balance.
2. Call `take_offer`.
3. Pre-fix: the incorrect invariant triggers an `assert_eq!` panic.
4. Post-fix: instruction succeeds and invariants validate correctly.

### Issue 2 (rent capture)
1. Construct `take_offer` accounts such that `payer = taker` (or another taker-controlled account).
2. Pre-fix: closing the offer account refunds lamports to `payer`.
3. Post-fix: lamports are refunded to the maker.

## Verification
Run the repo tests:

```bash
cd tokens/escrow/native
pnpm -s build-and-test
```

On this host (2026-02-15), with `rustc 1.93.1` / `cargo 1.93.1`, the test run passes:

```
# tests 3
# suites 1
# pass 3
# fail 0
```

(Full local log archived at: `superteam/_local_test_run_escrow_native_2026-02-15.txt`)

## Notes
- PR #533 currently contains **only the escrow-native changes** (3 files under `tokens/escrow/native/...`).
  - If you see an automated PR summary mentioning `basics/transfer-sol/...`, that was from an earlier iteration and is now stale; the current PR head is `yukikm:fix/escrow-native-rent-refund`.
- The fix is proposed upstream as PR #533 (link above) and is also available as a public fork commit + git-format patch for easy review.
