# CI-aware install defaults

## Summary

When `npm install` runs in a CI environment, default to validating that `package.json` and `package-lock.json` are in sync, and refuse to mutate the lockfile if they are not. The detection uses `ci-info`, which npm already ships. A new `ci-install` config controls the behavior, with `--no-ci-install` available as the escape hatch. `npm install <pkg>`, `npm install --package-lock-only`, and `npm update` are not affected. `npm ci` is not affected.

## Motivation

`npm install` in CI is non-deterministic in a way that hurts reproducibility and security, and the one command that fixes it (`npm ci`) is opt-in. That's the whole problem in one sentence.

### Why "tell users to use `npm ci`" is not enough

`npm ci` has existed since npm 5.7.0. The advice to use it in CI is in [npm's own docs](https://docs.npmjs.com/cli/v11/commands/npm-ci), in the [OWASP NPM Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html), in [OpenSSF Scorecard](https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies), and in countless tutorials. However, the fraction of CI workflows that actually use it is not what it should be. The reason is the same reason most opt-in security features underperform: the dangerous default is what people reach for, because it's what they type locally.

Both pnpm and Yarn looked at this same evidence years ago and made frozen installs the CI default. pnpm has done so since at least pnpm 6 ([docs](https://pnpm.io/cli/install#--frozen-lockfile)). Yarn flipped `enableImmutableInstalls` to default `true` on CI in Yarn 3.0, December 2021 ([CHANGELOG](https://github.com/yarnpkg/berry/blob/master/CHANGELOG.md#300)). Bun and Deno have the flag, `--frozen-lockfile` and `--frozen` respectively, but not the default ([Bun](https://bun.sh/docs/cli/install), [Deno](https://docs.deno.com/runtime/reference/cli/install/)).

## Detailed Explanation

### Behavior matrix

| Invocation | `ci-install=true` (CI default) | `ci-install=false` (local default) |
| --- | --- | --- |
| `npm install`, lockfile in sync | Install proceeds. No mutation. | Install proceeds. Same as today. |
| `npm install`, lockfile out of sync | Non-zero exit with diff. | Install proceeds, lockfile mutated. Same as today. |
| `npm install`, no lockfile at all | Install proceeds, lockfile created. | Install proceeds. Same as today. |
| `npm install <pkg>` | Bypassed. Lockfile updated as today. | Same as today. |
| `npm install --package-lock-only` | Bypassed. | Same as today. |
| `npm install --no-ci-install` | Forced off. Same as today. | Same as today. |
| `npm update` | Bypassed. | Same as today. |
| `npm ci` | Unchanged. | Unchanged. |

The "no lockfile present" case matters because container image builds and first-time installs in CI are common, and breaking them would be a non-starter. This RFC follows pnpm's `frozenLockfileIfExists` pattern, which only enforces frozen behavior when a lockfile actually exists on disk. Yarn made the opposite choice and errors out; the migration pain there is real, and I'd rather avoid it.

### CI detection

`ci-info`'s `isCI` boolean. The well-known escape hatch, `CI=false` in the environment, already disables all detection in `ci-info`, so anyone with a quirky local setup that trips `isCI` has a one-line fix.

### What this RFC does not change

- `npm ci` semantics.
- `npm install <pkg>` (with positional args). The whole point of that invocation is to add or modify deps.
- `npm install --package-lock-only`. Lockfile-only regeneration is by definition a mutation.
- `npm update`.
- Lockfile format.

## Rationale and Alternatives

### Alternative A: Do nothing, rely on docs pointing to `npm ci`

Status quo for seven years. Hasn't worked. I don't think another seven will be different.

### Alternative B: Add an explicit `--frozen-lockfile` flag, no auto-detection

This is what Bun and Deno chose, and effectively what [npm/rfcs#415](https://github.com/npm/rfcs/issues/415) proposes under the name `--from-lockfile`. The benefit is zero behavior-by-default change. The cost is that it perpetuates the exact opt-in problem this RFC exists to solve. A CI-using developer would need to know the flag exists, remember to use it, and put it in every workflow. We already have that command. It's `npm ci`. A flag does less.

### Alternative C: Auto-detect CI and switch to full `npm ci` semantics, including `node_modules` wipe

The wipe is a significant slowdown for projects that cache `node_modules`, and it's strictly stronger than what closes the threat model. Validating that the lockfile is in sync is sufficient. Deleting `node_modules` is a separate property. Keeping the wipe as a feature of the explicit `npm ci` command keeps the two intents distinct. This also matches the design of `pnpm install --frozen-lockfile`, `yarn install --immutable`, and `bun install --frozen-lockfile`: none of them imply `node_modules` deletion.

### Alternative D: Auto-detect CI with no new config key (no escape hatch)

Cleaner config surface, but no escape hatch is a bad idea. Some workflows intentionally mutate the lockfile in CI: release automation that bumps a version, self-hosted runners with idiosyncratic env vars, workflows that intentionally regenerate before commit. The escape hatch needs to exist, and it should be discoverable as a config (`--no-ci-install`), not as an opaque env var manipulation.

### Alternative E: Use a non-standard env var (`NPM_FROZEN_LOCKFILE=1`) instead of `ci-info`

`ci-info` is already a bundled dependency, is already used in five other places in the CLI, and is the convergent choice across npm, pnpm, Yarn, and Bun. Introducing a new npm-specific env var fragments the ecosystem when an existing convention already works.

## Implementation

The change is small. `npm install` (with no positional args) reuses the existing `loadVirtual` -> `buildIdealTree` -> `validateLockfile` sequence that `lib/commands/ci.js` already runs, and throws with an `ECIINSTALL` code (and an actionable message pointing at `--no-ci-install`) when the lockfile is out of sync. A new `ci-install` config of type `[null, Boolean]` is added to the existing `workspaces/config/lib/definitions/definitions.js`. When the value is `null`, the flatten function resolves it via `require('ci-info').isCI`. `ci-info` is already a bundled dependency, already imported in that file, and already used to drive the `progress` default and the `user-agent` CI vendor tag. `npm ci`, `npm update`, `npm install <pkg>`, and `npm install --package-lock-only` are all left alone.

Tests need to cover, at minimum:

- Lockfile-in-sync with `CI=true` succeeds
- Lockfile-out-of-sync with `CI=true` fails with the new error code
- No lockfile present with `CI=true` proceeds and creates one
- `--no-ci-install` overrides
- Positional args and `--package-lock-only` bypass the check
- `--no-package-lock` short-circuits cleanly

### Migration

The proposed rollout follows the existing precedent: config introduced in a feature version as opt-in, default flipped one major later.

| Release | Change |
| --- | --- |
| Next minor release | Land `ci-install` config with `default: null` and CLI flag. When `ci-install` is unset *and* CI is detected *and* the install would mutate the lockfile, emit a warning: `npm WARN ci-install A future major version of npm will treat this as an error. Pass --no-ci-install to opt out, or update your lockfile.` Behavior is otherwise unchanged. |
| Next major release | Flip the default. Warning becomes error. Migration guide in release notes. |

## Prior Art

| Tool | Mechanism | Default in CI | Detection | Lockfile required? |
| --- | --- | --- | --- | --- |
| pnpm | `frozen-lockfile` | `true`, if lockfile present | `ci-info` | Yes if present |
| Yarn Berry | `enableImmutableInstalls` | `true` | `ci-info` | Hard error if absent |
| Bun | `--frozen-lockfile` flag, `bun ci` command | Never auto | Custom port of `ci-info@4.0.0` | n/a |
| Deno | `--frozen[=bool]` flag, `deno.json` `"lock": {"frozen": true}` | Never auto | None | n/a |
| npm today | `npm ci` (separate command) | Explicit only | None | Yes (`npm ci` errors if missing) |

The closest prior npm proposal is [npm/rfcs#415](https://github.com/npm/rfcs/issues/415) (open since August 2021), which proposes an explicit `--from-lockfile` flag. The single substantive maintainer response framed any `npm install` / lockfile divergence as a bug to fix rather than a default to harden against. This RFC's position is that hardening the default is a separate concern from fixing the bugs, and that both should happen.

Yarn 3.0's CHANGELOG entry for the same change is worth quoting because it's almost exactly the change this RFC proposes, in the same ecosystem, and there's a multi-year track record of it working:

> The `enableImmutableInstalls` will now default to `true` on CI (we still recommend to explicitly use `--immutable` on the CLI). You can re-allow mutations by adding `YARN_ENABLE_IMMUTABLE_INSTALLS=false` in your environment variables.

## Unresolved Questions and Bikeshedding

1. **Config key name.** `ci-install` is one option. Others worth considering:
   - `frozen-lockfile` matches pnpm and Bun, but risks confusion because it doesn't include the `node_modules` wipe people might expect from prior `npm ci` usage.
   - `immutable` matches Yarn 2+, scopes are clearer.
   - `strict-lockfile` is npm-native and has no cross-PM collision.

   I lean `ci-install` because it describes the trigger as much as the behavior, but I have no strong attachment.

2. **CLI flag aliasing.** Should `--frozen-lockfile` and `--immutable` be accepted as aliases for `--ci-install`? Cross-package manager muscle memory is real.

3. **Exit code.** `npm ci` failures exit 1 with a usage error. Should `ci-install` failures use the same code, or a distinct one so CI systems can distinguish "lockfile drift" from "everything else"?

4. **`--package-lock-only` bypass: implicit or explicit?** This RFC bypasses it implicitly. The alternative is requiring the user to pass `--no-ci-install` when combining with `--package-lock-only`. Implicit aligns with pnpm. Explicit is more conservative. I lean implicit. Bots shouldn't need a config change.
