# solana-mobile-dapp-scaffold: yarn audit remediation (template deps)

# Security fix: patch vulnerable Solana Mobile dApp Scaffold dependencies (@solana/web3.js + transitive)

Repo: https://github.com/solana-mobile/solana-mobile-dapp-scaffold

- Baseline commit audited: `ed4dac7e30791d706feabd1f3d180663ee2e3d25`
- Fix commits: `e76f6b22bc0235d56e22b99b2b4cd4d454fd97d0` + `17d1f863c5de4f31c0fba2ec4d20bdc67b8322ab`
- Combined patch file: `superteam/solana-mobile-dapp-scaffold-fix-v2.patch`

## Summary
The template app under `template/` depended on **@solana/web3.js < 1.75.1** and transitive dependencies (`ws`, `base-x`) that were flagged by `yarn audit` with high severity issues.

This matters for Solana Mobile template users because:
- `@solana/web3.js` is used for RPC connections and websocket subscriptions.
- Many developers will ship this template directly into production bots/dApps.
- Vulnerable websocket and base58 tooling has a history of being exploited via DoS / parser edge cases / lookalike attacks.

## Impact
### 1) @solana/web3.js advisory (High)
`yarn audit` reported a high severity issue in `@solana/web3.js`:

- Advisory: https://www.npmjs.com/advisories/1097075
- Description (npm): “Handling untrusted input can result in a crash, leading to loss of availability / denial of service”
- Fix: upgrade to `@solana/web3.js >= 1.75.1`

Practical impact: apps that process attacker-controlled RPC data (or untrusted account data passed into parsing helpers) could be crashed (DoS) in the JS runtime, potentially taking down a mobile client or backend service.

### 2) `ws` advisory (High)
`yarn audit` reported `ws` DoS issues:

- Advisory: https://www.npmjs.com/advisories/1098393
- Fix: upgrade to a patched `ws` (the lockfile resolves to `8.17.1` in this fix)

Practical impact: even if an app doesn’t expose a websocket server, many dev tools and RPC websocket clients use `ws`. Keeping it patched reduces exposure to known header-based DoS conditions.

### 3) `base-x` advisory (High)
`yarn audit` reported a homograph/lookalike issue:

- Advisory: https://www.npmjs.com/advisories/1104178
- Fix: `base-x >= 4.0.1`

Practical impact: Base58 (Solana addresses, signatures) is a user-facing surface. Homograph-related issues can enable address spoofing / confusion in UI or logs.

## Reproduction
From repo root:

```bash
cd template
yarn audit --groups dependencies
```

### Before (proof excerpt)
`yarn audit` reports (excerpt):

- `Package       │ @solana/web3.js`
- `Patched in    │ >=1.75.1`
- `More info     │ https://www.npmjs.com/advisories/1097075`

And `base-x` (excerpt):

- `Package       │ base-x`
- `Patched in    │ >=4.0.1`
- `More info     │ https://www.npmjs.com/advisories/1104178`

Overall summary line at the end:
- `78 vulnerabilities found - Packages audited: 707`

### After (proof)
After applying the combined patch (`solana-mobile-dapp-scaffold-fix-v2.patch`), these advisories are no longer present in `yarn audit` output, and the end-of-run summary is:

- `64 vulnerabilities found - Packages audited: 709`

Not all remaining issues are addressed in this patch because many originate from pinned React Native tooling; upgrading RN is a larger breaking change and out of scope for a minimal, safe patch.

## Fix
Changes:
1. Bump `@solana/web3.js` in `template/package.json` from `^1.74.0` to `^1.75.1`.
2. Add Yarn `resolutions` to ensure patched transitive versions are selected:
   - `ws`: `^8.17.1`
   - `base-x`: `^4.0.1`
3. Regenerate `template/yarn.lock` via `yarn install`.

## Verification
1. `yarn audit --groups dependencies` no longer reports:
   - advisory 1097075 (`@solana/web3.js`)
   - advisory 1098392 (`ws` via rpc-websockets)
   - advisory 1104177 (`base-x`)
2. App still builds dependency graph successfully with Yarn classic.

## Patch / PR
I prepared a clean git commit and an email-style patch:

- Commit: `e76f6b2` (see workspace patch file)
- Patch: `superteam/solana-mobile-dapp-scaffold-e76f6b2.patch`

If maintainers want a PR, this commit can be pushed as a branch `security/bump-web3js-1.75.1`.

## Notes / follow-ups
- Remaining `yarn audit` findings are mostly due to the old `react-native@0.71.4` toolchain. Addressing them likely requires upgrading RN and Metro, which is a larger coordinated change.
