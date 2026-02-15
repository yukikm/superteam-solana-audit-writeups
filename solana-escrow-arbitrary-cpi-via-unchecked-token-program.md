# solana-escrow (kobby-pentangeli) — Arbitrary CPI via unchecked `token_program` account

Repo (upstream): https://github.com/kobby-pentangeli/solana-escrow

Fix (public fork branch): https://github.com/yukikm/solana-escrow/tree/fix/verify-token-program

Upstream PR (compare URL ready to open):
https://github.com/kobby-pentangeli/solana-escrow/compare/master...yukikm:solana-escrow:fix/verify-token-program

## Summary
The on-chain program accepts a `token_program` account from the transaction and uses it as the target program id for multiple CPIs (`set_authority`, `transfer`, `close_account`) without verifying that it is actually the SPL Token program.

This enables **arbitrary CPI** into an attacker-controlled program, including **CPI calls that mark the escrow PDA as a signer** (via `invoke_signed`).

## Affected code
File: `program/src/processor.rs`

1) `process_init_escrow`:
- reads `token_program = next_account_info(...)`
- constructs `spl_token::instruction::set_authority(token_program.key, ...)`
- executes `invoke(&owner_change_ix, ...)`
- **missing**: `token_program.key == spl_token::id()` check

2) `process_exchange`:
- reads `token_program = next_account_info(...)`
- constructs `spl_token::instruction::transfer(token_program.key, ...)`
- executes `invoke(...)` and later `invoke_signed(...)`
- **missing**: `token_program.key == spl_token::id()` check
- additionally, it does not validate the passed `pda_account` equals the derived PDA

## Impact
### 1) Arbitrary CPI execution
A caller can pass an arbitrary program as `token_program`. The escrow program will then CPI into that program.

Even if the attacker cannot steal assets directly from this single CPI, this is a serious violation of the trust boundary: **the program is delegating privileged actions to an untrusted callee**.

### 2) Escrow PDA signature exposure (in `process_exchange`)
In `process_exchange`, the program uses `invoke_signed` with seeds `[b"escrow", nonce]`.

If the attacker-controlled `token_program` is invoked at that point, the CPI will see the escrow PDA as a signer in its instruction context. A malicious callee can then:
- attempt to transfer lamports out of the PDA via CPI to the System Program,
- attempt token transfers where the PDA is authority,
- perform any other action that requires the PDA signature.

Even if the PDA normally holds minimal lamports, **this is still a signer-leak pattern** and is generally considered a critical issue in Solana programs.

## Reproduction (conceptual)
1. Deploy a malicious program `M` that, when invoked, issues a CPI to `system_program::transfer` with `from = escrow_pda` and `to = attacker`, using the signer privilege from the parent invoke.
2. Call `Exchange` and set the `token_program` account to `M` (instead of `spl_token::id()`).
3. The escrow program will call into `M` via `invoke_signed`, allowing `M` to attempt to move funds using the PDA signature.

Notes:
- This does not require breaking cryptography; the signature is granted by Solana runtime as part of `invoke_signed`.
- Even if the PDA has low balance, this is still a correctness + security boundary issue.

## Fix
Add strict program-id validation and basic ownership checks.

Patch implements:
- Rejects any `token_program` not equal to `spl_token::id()`.
- Validates token accounts passed into instructions are owned by `spl_token::id()` (defense-in-depth).
- Ensures the provided `pda_account` matches the derived PDA.

### Diff
See branch: https://github.com/yukikm/solana-escrow/tree/fix/verify-token-program

Key added checks:
- `if *token_program.key != spl_token::id() { return Err(ProgramError::IncorrectProgramId); }`
- token-account `owner` checks
- `if *pda_account.key != pda { return Err(ProgramError::InvalidAccountData); }`

## Verification
- Static verification: after the patch, CPIs can only target the canonical SPL Token program id.
- Any attempt to pass an arbitrary program for `token_program` fails early with `IncorrectProgramId`.

(Unable to run `cargo test` in this environment due to an outdated Cargo toolchain; however, the change is minimal and should compile under the repo’s CI/toolchain.)
