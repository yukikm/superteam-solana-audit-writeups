# Anchor IDL spec: malformed array types could panic the parser (DoS)

## TL;DR
In `anchor-lang-idl-spec`, parsing some malformed array type strings (e.g. `"[u8 32]"`) triggered a **panic** due to an `unwrap()` on a missing `;`. This can be abused by feeding a toolchain a malicious / corrupted IDL to **crash** the process (denial-of-service) instead of returning a normal error.

I patched the parser to return a structured `anyhow::Error` rather than panicking, and I added regression tests.

- Repo: https://github.com/solana-foundation/anchor
- Affected crate/path: `idl/spec/src/lib.rs` (`anchor-lang-idl-spec`)
- Fix commit (local): `0cd59ba4e0e6b2b8b0f9f4a1d1f9f0c8d3f0d7b1` (see note below)
- Patch file: `superteam/anchor-idl-array-no-panic.patch`

> Note: the commit hash above is from my local clone (based on Anchor `a95c356415c203ea61bb3574765a7f60a542c640`). If you apply the patch on a different commit, your resulting hash will differ.

---

## Impact
The `IdlType::from_str()` parser is used to interpret IDL type strings. Prior to this fix, malformed array syntax could trigger a panic:

- Missing semicolon between element type and length: `"[u8 32]"`

This is a denial-of-service class issue: any application that parses untrusted/third-party IDLs (or even just accidentally malformed IDLs) could crash, yielding poor UX and potentially breaking CI/build pipelines.

---

## Root cause
In `idl/spec/src/lib.rs`, the array parsing helper did:

```rust
let (raw_type, raw_length) = inner.rsplit_once(';').unwrap();
```

If `inner` does not contain `;`, `rsplit_once` returns `None` and `unwrap()` panics.

Additionally, the parser used `IdlType::from_str(raw_type).unwrap()` which would also panic if the element type string is syntactically invalid (e.g. missing `>` in `Option<...>`). While unknown type *names* are allowed (they become `IdlType::Defined`), syntactically invalid forms can produce an `Err`, which previously could also panic.

---

## Reproduction
### 1) Minimal repro (before fix)
Run this snippet in any binary that depends on `anchor-lang-idl-spec`:

```rust
use anchor_lang_idl_spec::IdlType;
use std::str::FromStr;

fn main() {
    // Panics pre-fix
    let _ = IdlType::from_str("[u8 32]");
}
```

Expected (bad) behavior pre-fix: panic with something like `called Option::unwrap() on a None value`.

### 2) Post-fix behavior
After applying the patch, the same input returns a normal error:

- `IdlType::from_str("[u8 32]")` → `Err(...)`

---

## Fix
Change the helper to return `Result<IdlType, anyhow::Error>` and replace the `unwrap()` calls with explicit error propagation:

- `rsplit_once(';').ok_or_else(|| anyhow!(...))?`
- `IdlType::from_str(raw_type).map_err(|e| anyhow!(...))?`

This preserves existing behavior for valid array types and nested arrays, but safely rejects malformed input.

---

## Verification
I added two regression tests in `idl/spec/src/lib.rs`:

- `malformed_array_missing_semicolon_returns_err`
- `malformed_array_invalid_element_type_returns_err`

To run:

```bash
cargo test -p anchor-lang-idl-spec
```

All tests pass after the fix.

---

## Patch application instructions
From a clean Anchor checkout:

```bash
git checkout master
git apply /path/to/anchor-idl-array-no-panic.patch
cargo test -p anchor-lang-idl-spec
```

---

## Disclosure notes
This is a safe, minimal change (no new dependencies, no behavior changes on valid input) and is appropriate for upstreaming as a bugfix.
