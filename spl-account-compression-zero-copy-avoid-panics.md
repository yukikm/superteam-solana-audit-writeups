# SPL Account Compression: avoid panics in `ZeroCopy::{load_bytes,load_mut_bytes}`

Repo: https://github.com/solana-labs/solana-program-library

Component: `account-compression/programs/account-compression`

Patch: `superteam/solana-program-library-account-compression-zero-copy-no-unwrap.patch`

Commit (local): `c32ef61` (branch: `fix/account-compression-zero-copy-no-unwrap`)

## Summary
`ZeroCopy::{load_bytes, load_mut_bytes}` used `unwrap()` and performed an unchecked slice (`&data[..size]`). If a caller passes an account whose data length is **smaller than the expected type size**, the program **panics** (Rust panic) instead of returning a normal Solana/Anchor error.

On Solana, panics are undesirable because they:
- hard-fail the instruction (DoS-style reliability issue),
- can make failures non-obvious to integrators,
- bypass the program’s intended error handling (`AccountCompressionError::ZeroCopyError`),
- and in general violate the “never panic on user-controlled input” best practice.

The fix adds explicit length checks and removes the `unwrap()` calls, returning `ProgramError::InvalidAccountData` in the underlying helper, which is then surfaced by the existing macro layer as `AccountCompressionError::ZeroCopyError`.

## Affected code
File:
- `account-compression/programs/account-compression/src/zero_copy.rs`

Before:
- `bytemuck::try_from_bytes(_).map_err(...).unwrap()`
- slicing `data[..size]` without checking `data_len >= size`

## Impact
**Any instruction path that calls these helpers with an undersized byte slice can be made to panic.**

Because Solana instructions can be invoked with arbitrary accounts, an attacker can pass in a deliberately undersized account in place of a “tree account” / data account if the account constraints do not strictly enforce size, causing a panic rather than a handled error.

Even when the panic only affects the attacker’s own transaction, removing panics is valuable:
- improves program robustness,
- ensures predictable error codes for clients,
- avoids undefined behavior risk from assuming layout/alignment without checks.

## Reproduction (conceptual)
1. Create a new account with **very small data length** (e.g. 0 bytes or a few bytes).
2. Call an `spl-account-compression` instruction that ends up invoking `ConcurrentMerkleTree::<...>::load_bytes` or `load_mut_bytes` on the account’s data buffer.
3. Before the fix: the program panics due to `&data[..size]` out-of-bounds and/or `unwrap()`.
4. After the fix: the call returns a normal program error (invalid account data / zero-copy error) instead of panicking.

A direct unit-test reproduction is included in the patch (tests in `zero_copy.rs`) to show that a short slice now returns `Err` and does not panic.

## Fix
- Add `if data_len < size { ... return Err(ProgramError::InvalidAccountData.into()) }` guard.
- Replace `unwrap()` with `?` so bytemuck conversion failures are handled.
- Add unit tests ensuring short slices return an error rather than panicking.

## Verification
Run:

```bash
cd account-compression/programs/account-compression
cargo test
```

Result (local):
- `test result: ok. 32 passed; 0 failed` (includes new tests)

## Suggested upstream PR
Title:
- `account-compression: avoid panics in ZeroCopy helpers`

Body:
- explain panic risk, link this write-up, mention `cargo test`.
