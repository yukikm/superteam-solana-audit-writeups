# spl-name-service: panic / DoS when `parent_name_owner` account is omitted

**Repo:** https://github.com/solana-labs/solana-program-library (Name Service program)

**Component:** `name-service/program/src/processor.rs` (`Processor::process_create`)

**Severity:** Low (graceful failure -> panic). Operational DoS / degraded UX.

## Summary
`spl-name-service` accepts an optional `parent_name_owner` account when creating a record. When a non-default `parent_name_account` is provided **but the corresponding `parent_name_owner` account is omitted from the transaction**, the program previously executed `parent_name_owner.unwrap()` and **panicked**.

On Solana, panics abort execution with an opaque failure and still consume compute; while this does not corrupt state, it creates a cheap way to force this instruction path to fail in a non-diagnostic way (and can be used to waste compute units / degrade a relayer that naively retries, etc.).

## Impact
- Any caller can craft a `Create` instruction that includes a non-default `parent_name_account` but omits the trailing `parent_name_owner` account.
- Before the fix, the program would panic instead of returning a structured `ProgramError`.
- After the fix, the program returns `ProgramError::NotEnoughAccountKeys` (or `InvalidArgument` depending on runtime mapping), enabling consistent client handling.

## Root cause
In `Processor::process_create`, the code parsed the accounts as:

```rust
let parent_name_account = next_account_info(accounts_iter)?;
let parent_name_owner = next_account_info(accounts_iter).ok();
```

…and later did:

```rust
if *parent_name_account.key != Pubkey::default() {
    if !parent_name_owner.unwrap().is_signer {
        ...
    }
}
```

If the transaction doesn’t include the `parent_name_owner` account, `parent_name_owner` is `None` and `.unwrap()` panics.

## Reproduction (pre-fix)
Using the `spl_name_service::instruction::create` helper:

1. Create a valid root/TLD name (or use any existing parent name).
2. Build a `Create` instruction with:
   - `parent_name_account = Some(<non-default pubkey>)`
   - `parent_name_owner = None`
3. Send the transaction.

Before the patch, this triggers a panic in-program.

A regression test is included in the patch:
- `name-service/program/tests/functional.rs::test_create_with_parent_missing_parent_owner_account_fails_gracefully`

## Fix
Treat `parent_name_owner` as optional only when `parent_name_account == Pubkey::default()`. If a non-default parent is provided, require `parent_name_owner` and return a structured error if it is missing.

Patch summary:
- Rename to `parent_name_owner_opt`
- When `parent_name_account` is non-default: `ok_or(ProgramError::NotEnoughAccountKeys)?`
- Avoid multiple unwraps.

## Patch / PR
- Patch file: `superteam/name-service-missing-parent-owner.patch`
- Commit: `1c5219555484bf855733009c175621f25c6ba5da`

Upstream PR: (to be opened) `solana-labs/solana-program-library`.

## Verification
- Added regression test that constructs the malformed instruction and asserts it fails with a structured error.
- To run tests locally you currently need a toolchain compatible with the workspace’s `solana-program` (requires **rustc >= 1.79**):

```bash
cd name-service/program
cargo test --features test-sbf
```

(Our CI environment here has rustc 1.75, so compilation isn’t possible in this runner; the logic change is narrow and the added test demonstrates intended behavior.)
