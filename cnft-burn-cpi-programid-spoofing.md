# `cnft-burn` example: CPI program-id spoofing (malicious program hijack)

**Repository:** https://github.com/solana-developers/program-examples

**Path:** `compression/cnft-burn/anchor` (Anchor program `cnft-burn`)

**Type:** Unchecked CPI program account (program-id spoofing)

**Severity:** High (signer privilege can be redirected to attacker-controlled program)

## Summary
The `burn_cnft` instruction performs a CPI using a `bubblegum_program` account **provided by the caller**, but the program did not constrain that account to the canonical Bubblegum program id (`mpl_bubblegum::ID`).

Because `leaf_owner` is forwarded as a **signer** in the CPI, a malicious caller can supply an attacker-controlled program id as `bubblegum_program`, causing the CPI to invoke that malicious program with `leaf_owner` as an authorized signer.

This is a common Solana/Anchor security pitfall: **using an unvalidated program account as the CPI target**.

## Impact
If a client/UI integrates this example and passes accounts “as provided”, a malicious transaction can be crafted where:

- `bubblegum_program` is **not** the Bubblegum program.
- The CPI is redirected to attacker code.
- Attacker code can use `leaf_owner` signer privilege to do unauthorized actions (e.g., `system_program::transfer` draining SOL).

Although this is an **example** repo, these patterns are frequently copy‑pasted into production programs.

## Root cause
In `BurnCnft` accounts (before fix), the following were not constrained:

- `bubblegum_program: UncheckedAccount<'info>` (no `address = mpl_bubblegum::ID`)
- `log_wrapper: UncheckedAccount<'info>` (no `address = spl_noop::id()`)
- `merkle_tree` ownership (no `owner = spl_account_compression::id()`)

The instruction then constructs and invokes a CPI:

```rust
let cpi = mpl_bubblegum::instructions::BurnCpi::new(
    &ctx.accounts.bubblegum_program,
    ...
);

cpi.invoke_with_remaining_accounts(...)?;
```

If `bubblegum_program` is not constrained, the CPI target is attacker-controlled.

## Exploit sketch (minimal)
1. Attacker deploys a malicious program `P` that takes `leaf_owner` as a signer and executes a `system_program::transfer` from `leaf_owner` to an attacker address.
2. Attacker constructs a transaction that calls `burn_cnft`, but sets:
   - `bubblegum_program = P`
   - `tree_authority` to the PDA derived using `P` (the example derives the PDA using `seeds::program = bubblegum_program.key()`)
3. User signs the transaction believing it is a normal Bubblegum burn.
4. The program CPIs into `P`, which drains the user.

## Fix
Constrain CPI program accounts and validate merkle tree ownership:

- Constrain `bubblegum_program` to `mpl_bubblegum::ID`.
- Constrain `log_wrapper` to `spl_noop::id()`.
- Constrain `merkle_tree` to be owned by `spl_account_compression::id()`.

This guarantees the CPI can only target the intended programs.

### Patch (public)
- Fix commit (fork): https://github.com/yukikm/program-examples/commit/be61248
- Branch: `fix/cnft-burn-validate-cpi-programs`
- PR creation link (upstream): https://github.com/solana-developers/program-examples/compare/main...yukikm:program-examples:fix/cnft-burn-validate-cpi-programs?expand=1

## Verification
A negative test was added which passes a bogus `bubblegumProgram` public key and asserts the transaction fails with an Anchor `ConstraintAddress` error.

File: `compression/cnft-burn/anchor/tests/cnft-burn.ts`

## Notes on reproducibility
In this environment, I could not run `anchor build/test` because the `anchor` CLI is not installed and the system `cargo` is too old for a dependency requiring the stabilized `edition2024` feature.

The fix is small and should be validated in a standard Solana/Anchor dev environment (or CI) via:

```bash
cd compression/cnft-burn/anchor
anchor test
```
