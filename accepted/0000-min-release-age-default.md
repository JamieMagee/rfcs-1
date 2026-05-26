# Set min-release-age default to three days

## Summary

Change the default of `min-release-age` from `null` to `3`. npm will skip versions published in the last 72 hours unless the user opts out with `min-release-age=0`. The default applies only to `registry.npmjs.org`; private and self-hosted registries are unaffected. `npm audit fix` keeps installing patches for reviewed advisories inside the cooldown window via a scoped, observable bypass.

## Motivation

When a maintainer credential is compromised, the dangerous window is short. The malicious version is live, CI and update tools pick it up immediately, and detection takes hours to a few days. `min-release-age` already adds that delay, but it is off by default and almost no one sets it.

The published evidence is small but consistent. William Woodruff's November 2025 dataset[^woodruff] catalogued ten supply-chain compromises across 2024-2025 with their publication-to-takedown windows. Seven were npm- or PyPI-side fast-burn attacks (chalk, Nx, rspack, web3.js, Ultralytics phases 1 and 2, num2words) with windows of one to twelve hours; a 24-hour cutoff catches those. One ran approximately three days (`tj-actions/changed-files`, a GitHub Actions compromise, used here as instructive precedent rather than an in-scope case); 72 hours catches it, 24 hours does not. No incident in the dataset is caught at 7 days that is missed at 72 hours.

72 hours is a bracketed choice rather than a value the data proves optimal. Three days covers the multi-day fast-burn windows that 24 hours misses, and the dataset shows no incremental coverage from extending past three days. The dataset is too small to bear precise prevention-rate claims; this is a policy proposal informed by the data we have.

This is not a complete supply-chain defense. It does not detect malicious code, does not help when a malicious package goes undetected for weeks (as with `event-stream`[^event-stream]), and does not re-check what is already in a lockfile. The goal is narrower: avoid being among the first installs of a just-published version on the public registry. All five major JavaScript package managers shipped a cooldown feature in the last six months, and Dependabot, Renovate, and Snyk ship default or recommended cooldowns at the layer above. See [Prior Art](#prior-art).

## Detailed Explanation

`min-release-age` wraps the existing `before` option. With `min-release-age=3`, the cutoff is the client's wall clock minus three days (259,200 seconds), and npm only resolves versions published before that cutoff. Registry publish times remain the source for package age. Minor client clock skew is acceptable.

The default becomes:

```ini
min-release-age=3
```

Users opt out at any config level:

```sh
npm install --min-release-age=0
npm config set min-release-age 0
```

`0` is the supported off value. Documentation and error messages should use `min-release-age=0`, not `--no-min-release-age`.

### Scope: public registry only

The default applies only to `registry.npmjs.org`. Internal and private registries (GitHub Packages, AWS CodeArtifact, Artifactory, Verdaccio, on-prem mirrors) are not subject to it.

A public-registry publish is an upload by a credential the registry cannot fully attest. An internal-registry publish is the output of a reviewed pipeline tied to a code review and a release process. Applying a 72-hour delay to internal pipelines would break the canary-publish loop on every monorepo that depends on it, with no security benefit.

- Resolutions against `registry.npmjs.org` apply the default cutoff.
- Resolutions against any other configured registry (`@scope:registry=...`, `registry=...`, project-level overrides) are exempt.
- Users can opt in per-scope with `@scope:min-release-age=N`, or use the per-package exclude mechanism in [npm/cli#8994](https://github.com/npm/cli/issues/8994) once that ships.

### Where the filter applies

Anywhere npm resolves a version from `registry.npmjs.org` metadata: `npm install`, `npm install <pkg>`, `npm install <pkg>@latest`, `npm install <pkg>@1.2.3`, `npm update`, `overrides`, and `npm exec` / `npx <pkg>@latest`.

For range requests, npm may pick an older satisfying version. If `pkg@1.2.3` is too new and `pkg@1.2.2` also satisfies `^1.2.0`, npm should pick `1.2.2`.

Explicit tag and exact-version requests must not silently downgrade. `pkg@latest` pointing at a too-new version should fail rather than fall back. `pkg@1.2.3` that is too new should fail.

### Errors and visibility

Errors must name the package and version, show the publish time and cutoff, and include the exact opt-out flag:

```txt
pkg@1.2.3 was published at 2026-05-25T12:00:00.000Z,
which is newer than the min-release-age cutoff 2026-05-22T12:00:00.000Z.
To install it anyway, rerun with --min-release-age=0.
```

When npm picks an older version because newer candidates were filtered out, it should say so. A one-line summary is enough for normal output; verbose logs can include per-package filtered-candidate counts.

### Lockfiles

Existing lockfiles are authoritative. `npm ci` and any `npm install` whose lockfile already satisfies the graph install the locked versions without rechecking publish age. This is a resolution-time guardrail, not a lockfile scanner. New resolution is still filtered: `npm install <new-pkg>` against an existing lockfile, manifest edits that change dependencies, and partial lockfile drift.

### Missing publish time

1. If the packument has no `time` object at all, allow the version but warn that publish age could not be verified. This keeps private registries and mirrors working.
2. If the packument has a `time` object but no entry for the selected version, treat that version as invalid and reject it unless the user opts out. This prevents stripped per-version metadata from being a silent bypass.

### npm audit fix

`npm audit fix` applies a scoped bypass for versions selected to resolve a **reviewed entry in the GitHub Advisory Database** (`reviewed: true`). Versions that do not resolve a reviewed advisory remain subject to the cutoff.

- Bypass applies only when the selected version resolves a reviewed advisory with a concrete patched version or range. Unreviewed advisories and malware advisories without a patched range do not trigger automatic bypass.
- Bypass selects the **lowest non-vulnerable version** satisfying the project's semver constraints, not the newest available. This eliminates the "install the freshly-published 1.2.5 because 1.2.4 fixed the CVE" pattern.
- Every bypass emits a visible notice:

  ```txt
  min-release-age: bypassed for example@1.2.4 to resolve GHSA-xxxx-xxxx-xxxx (CVE-2026-12345)
  ```


A strict opt-out is available for organizations that prefer to delay even security updates: `min-release-age-strict=true`, mirroring pnpm's `minimumReleaseAgeStrict`.

The previous decision in [npm/cli#9212](https://github.com/npm/cli/issues/9212) was to apply the cooldown uniformly. That was coherent when the default was `null`. With a non-zero default, the same policy silently suppresses CVE patches, which defeats the purpose of `npm audit fix`. This RFC proposes reopening #9212 alongside the default change. Dependabot's `cooldown` already exempts security updates by default[^dependabot].

### npm audit signatures

`npm audit signatures` must not re-apply the cooldown filter against already-installed packages. The fix is mechanical: clear the `before` filter on the `pacote.manifest()` call inside `verify-signatures.js`. Tracked in [npm/cli#9277](https://github.com/npm/cli/issues/9277).

Without this fix, a team that adopts the cooldown today sees `npm audit signatures` produce spurious `ETARGET` errors on lockfile-pinned versions younger than the threshold. That is a CI breaker and a blocker for the default change.

### Out of scope

This feature only checks version publish time. It does not cover dist-tag movement; moving `latest` onto an older malicious version is still possible.

A per-package exclude list ([npm/cli#8994](https://github.com/npm/cli/issues/8994)) and a strict-metadata mode (failing closed on missing publish time) are both out of scope here; either can follow as a separate proposal.

## Rationale and Alternatives

### Keep the default at null

Avoids compatibility risk but leaves the feature opt-in, which is the situation today. The rate of supply-chain incidents and the convergence on cooldown defaults elsewhere argue against keeping the status quo.

### Use a 1-day default

A 24-hour cutoff matches pnpm 11 and catches the hour-scale fast-burn attacks that dominate the recent record. It also produces a smaller compatibility footprint. The reason we propose 3 days instead: published incidents with multi-day attacker-visible windows are not caught by 24 hours, and the marginal breakage cost between 24 and 72 hours is bounded because the protective surface is *new resolution*, not *every install* (see [Compatibility](#compatibility)). A 24-hour default would still be a real improvement over the status quo and is the obvious fallback if 72 hours proves too disruptive in Phase 1.

### Use a 7-day default

Catches more, breaks more. Renovate recommends 14 days for fully automated merging; that is a high cost in a default for `npm install`. Three days brackets most of the observed attack windows without a week-long install delay. The Phase 3 review can revisit this if the evidence supports it.

### Warning-only mode

Useful for measurement, not protection: it still installs the fresh version. A warning-only stage could precede this RFC but should not be the final behavior.

### Exempt exact-version requests

A compromised maintainer can publish an exact version just as easily as a range-compatible one. Users who need a fresh exact version can pass `--min-release-age=0`.

### Apply uniformly to all registries

A uniform 72-hour delay on internal pipelines would break the canary-publish loop on monorepos. The per-registry scope is necessary for a non-zero default.

### Apply the cooldown uniformly to `npm audit fix`

The position taken in the (currently closed) [npm/cli#9212](https://github.com/npm/cli/issues/9212). Coherent when the default is `null`; with a non-zero default it silently suppresses CVE patches, which is a worse failure mode than the (already mitigated) risk of an attacker publishing a malicious "fix". The scoped bypass is anchored to the GHSA `reviewed` flag and selects the lowest-eligible patched version.

### Add an exclude list now

A `min-release-age-exclude=@scope/*` option is useful but bigger than a default change. Discussed in [npm/cli#8994](https://github.com/npm/cli/issues/8994). Deserves a separate RFC.

### Add a strict-metadata mode now

A future `min-release-age-require-metadata` could let trusted-registry users fail closed on missing publish time. Out of scope here.

## Rollout

### Phase 0: resolve blockers

Ship the per-registry scope, the `npm audit fix` advisory bypass, and the `npm audit signatures` regression fix. Document each in release notes and a migration guide.

### Phase 1: opt-in beta

Recommend `min-release-age=3` in npm documentation, in `npm init`'s generated `.npmrc`, and in published security guidance. Evaluate impact through existing signals: GitHub issue volume tagged to cooldown behavior, the GHSA stream paired with npm's version-publication timestamps (a counterfactual for cooldown rewrites and suppressed advisory patches), and partner feedback from large internal-mirror operators. Registry-side `User-Agent` already tracks CLI-version adoption.

### Phase 2: default switch

Switch the default to `3`. Emit a single notice line at the default log level whenever a resolution is rewritten by the cooldown, including the package, the substituted version, and the override command. No silent rewrites.

### Phase 3: review

Evaluate whether to extend to 7 days based on:

- Published incidents in the prior 12 months whose windows fell between 72 hours and 7 days.
- The GHSA-versus-version-timestamp counterfactual: how often advisory-resolving versions were published inside a 7-day window, and how often the cooldown would have routed installs onto a known-vulnerable older version.
- Support and issue volume tagged to cooldown behavior.

## Implementation

Areas to touch in `npm/cli`:

- config: change `min-release-age` default to `3`; gate the default on the resolved registry URL.
- resolver and manifest picker: apply the cutoff to ranges, tags, exact specs, updates, overrides, and `npm exec` / `npx`.
- registry-scope check: skip the default for registries other than `registry.npmjs.org` unless the user opts in per-scope.
- audit fix planner: implement the scoped advisory bypass, with `reviewed: true` GHSA matching, lowest-patched-version selection, suspicious-candidate refusal, and a visible per-bypass notice. Coordinate with #9212.
- audit signatures: drop the `before` filter on `pacote.manifest()` calls against already-installed packages. Coordinate with #9277.
- error and summary output: per the format above.
- packument handling: distinguish whole-packument missing `time` from per-version missing entries.
- docs: document the public-registry scope and the `npm audit fix` bypass.

## Compatibility

`npm ci` and lockfile-satisfied `npm install` are unaffected. A cooldown affects resolution events, not raw install volume.

Fresh installs against `registry.npmjs.org` can change. If the newest satisfying version is less than three days old, npm picks an older satisfying version or fails when none exists. `npm install pkg@latest` can now fail if `latest` is too new; that is intentional.

Resolutions against other registries are unaffected by the default. Internal release pipelines that publish and immediately consume a package continue to work without configuration.

Pipelines that publish to the public registry and immediately consume the published version (the canary-publish-then-install pattern) should opt out for that step:

```sh
npm install pkg@latest --min-release-age=0
```

Private registries that omit publish times continue to work, with a warning that publish age could not be verified.

## Prior Art

All five major JavaScript package managers ship a cooldown feature, and all of them shipped it in the last six months.

- **pnpm** added `minimumReleaseAge` in v10.16[^pnpm-10.16] and defaulted it to 1440 minutes (24 hours) in v11[^pnpm-11]. pnpm also ships per-package exclude (`minimumReleaseAgeExclude`), a strict mode (`minimumReleaseAgeStrict`), and an ignore-missing-time toggle (default on). `pnpm audit --fix` writes the minimum patched version of each advisory to `minimumReleaseAgeExclude` so the fix installs immediately. This RFC's `npm audit fix` bypass takes a similar approach, scoped to reviewed advisories and gated on lowest-patched-version selection.
- **Yarn (Berry)** added `npmMinimalAgeGate` in v4.10.0 (September 2025)[^yarn]. Default 0.
- **Bun** added `--minimum-release-age` and a `minimumReleaseAge` setting in `bunfig.toml`[^bun] in v1.3 (October 2025). Default 0; the documentation uses a three-day example.
- **Deno** added `--minimum-dependency-age`[^deno] in 2025. Default 0.

Tooling layered above the package manager has converged more aggressively:

- **Dependabot** ships `cooldown`[^dependabot] with semver-level granularity and per-dependency include/exclude lists. Security updates bypass the cooldown by default.
- **Renovate** ships `minimumReleaseAge`[^renovate] and uses a 3-day npm cooldown in its best-practices preset. It recommends 14 days for fully automated merging.
- **Snyk** applies a non-configurable 21-day cooldown to its automated upgrade PRs (security PRs excluded).
- **npq** uses package age as an install-time signal[^npq].

Cross-language: uv, pip, and Poetry (Python) and Cargo (Rust, via RFC #3923) have shipped or are stabilizing cooldown features in the same window. Bundler (Ruby) and Go are in active design.

## Unresolved Questions

- Whether the per-registry scope should be a configurable knob (`min-release-age-public-only=true` or similar) or hard-wired to the npm public registry URL. We propose the latter for simplicity.
- The exact text of the bypass notice line. The format above is a starting point.
- Whether `--min-release-age=0` should be accepted (as a no-op) in `npm ci` for symmetry with `npm install`. Today it has no effect because lockfiles are authoritative.
- Whether to expose the strict opt-out (`min-release-age-strict=true`) in this RFC or defer to a follow-up. We propose including it so the bypass and the strict opt-out land together.

[^woodruff]: https://blog.yossarian.net/2025/11/21/We-should-all-be-using-dependency-cooldowns
[^event-stream]: https://blog.npmjs.org/post/180565383195/details-about-the-event-stream-incident
[^pnpm-10.16]: https://pnpm.io/blog/releases/10.16#new-setting-for-delayed-dependency-updates
[^pnpm-11]: https://pnpm.io/blog/releases/11.0#security--build-defaults
[^yarn]: https://github.com/yarnpkg/berry/releases/tag/%40yarnpkg%2Fcli%2F4.10.0
[^bun]: https://bun.sh/docs/cli/install#minimum-release-age
[^deno]: https://docs.deno.com/runtime/reference/cli/install/#--minimum-dependency-age
[^dependabot]: https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-
[^renovate]: https://docs.renovatebot.com/configuration-options/#minimumreleaseage
[^npq]: https://github.com/lirantal/npq
