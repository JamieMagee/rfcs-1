# Make install scripts opt-in

## Summary

Dependency install scripts (`preinstall`, `install`, `postinstall`, and auto-detected `node-gyp` builds) should be blocked by default during `npm install`. Projects opt in to running scripts for specific dependencies by listing them in a new `allowScripts` field in `package.json`. An interactive `npm approve-scripts` command helps users build and maintain this allowlist.

This proposal aligns npm with pnpm (v10+), Yarn Berry, and Bun, all of which already block dependency install scripts by default.

## Motivation

### The security case

Install scripts execute arbitrary shell commands the moment a package is installed. Unlike application code, which must be explicitly imported and run, install scripts fire automatically as a side effect of `npm install`.

Attackers have exploited this repeatedly:

- **event-stream (November 2018)**: A social engineering attack added a malicious `postinstall` dependency (`flatmap-stream`) targeting a specific Bitcoin wallet application.
- **Lazarus Group campaigns (2024-2025)**: North Korean state-sponsored actors ran sustained campaigns publishing typosquatted packages (`is-buffer-validator`, `react-event-dependency`, and others) with `postinstall` scripts that deployed the BeaverTail malware and InvisibleFerret backdoor, stealing browser credentials and cryptocurrency wallet data. [Source](https://www.bleepingcomputer.com/news/security/north-korean-lazarus-hackers-infect-hundreds-via-npm-packages/)
- **chalk, debug, and 17 other packages (September 2025)**: A phished maintainer account was used to inject Web3 wallet-draining code into 19 packages with over 2 billion combined weekly downloads. The payload was delivered via `postinstall` scripts in packages that had never needed install scripts before. [Source](https://www.ox.security/blog/npm-packages-compromised/)
- **Shai-Hulud worm (September 2025)**: A self-replicating `postinstall` payload compromised 500+ npm packages by stealing maintainer tokens and automatically publishing infected versions of the victim's other packages. CERT/CC issued advisory [VU#534320](https://www.kb.cert.org/vuls/id/534320). [Source](https://www.stepsecurity.io/blog/ctrl-tinycolor-and-40-npm-packages-compromised)
- **Axios (March 2026)**: Attackers hijacked the lead maintainer of Axios (100M+ weekly downloads) and published versions containing a "phantom dependency" that existed solely to trigger its `postinstall` hook, deploying a cross-platform RAT. The malicious package was never imported in Axios source code. [Source](https://www.trendmicro.com/en_us/research/26/c/axios-npm-package-compromised.html)

### Install scripts are a distinct threat class

A common response is "you're going to run the code anyway." But install scripts are different from application code in several ways that matter:

1. Install scripts run without any `require()` or `import`. They fire just by being in the dependency tree. This means they can attack systems that never intend to run the package's code at all: frontend-only projects, build machines, CI pipelines.

2. Install scripts may run under different privileges than the application. The widespread use of `--unsafe-perm` as a troubleshooting fix means many environments run install scripts as root.

3. Many organizations trigger `npm install` automatically in response to pull requests. Install scripts are often the only code execution surface on build machines that otherwise only compile and bundle.

4. A typo like `npm install canvsa` triggers the install script immediately, before the developer can notice the mistake and cancel. Without install scripts, the typosquatted package would sit inert until explicitly imported.

5. Adding a dependency to `package-lock.json` in a pull request is easy to miss: GitHub hides large lock file diffs by default, and few reviewers read them carefully. The install script runs the next time anyone runs `npm install`, without the attacker getting any application code reviewed.

6. Process sandboxing, import policies, and code review all target runtime code. None of them cover code that runs during installation.

### The ecosystem has moved on

When this RFC was first proposed in 2021, a common objection was that too many packages depend on install scripts, particularly native addons that compile via `node-gyp`. Since then, the ecosystem has shifted to prebuilt platform-specific binaries distributed via `optionalDependencies`:

- esbuild -> `@esbuild/linux-x64`, `@esbuild/darwin-arm64`, etc.
- SWC -> `@swc/core-linux-x64-gnu`, `@swc/core-darwin-arm64`, etc.
- Sharp -> `@img/sharp-linux-x64`, etc.
- Rollup -> `@rollup/rollup-linux-x64-gnu`, etc.
- lightningcss, Biome, oxc: all follow this prebuild pattern.

[Node-API](https://nodejs.org/api/n-api.html) provides ABI stability across Node.js versions, making prebuilt binaries viable without recompilation. The packages that still require install scripts are a small minority and getting smaller.

### npm is the last holdout

Every other major JavaScript package manager already blocks dependency install scripts by default:

| Package manager | Default behavior                                                   | Since                           |
|-----------------|--------------------------------------------------------------------|---------------------------------|
| pnpm            | Blocked; allowlist via `allowBuilds` in `pnpm-workspace.yaml`      | v10 (January 2025)              |
| Yarn Berry      | Blocked; per-package opt-in via `dependenciesMeta.built`           | v2 (2020)                       |
| Bun             | Blocked (except ~400 default-trusted); `trustedDependencies` array | Since install support was added |
| Deno            | Blocked; per-package opt-in via `--allow-scripts=<packages>`       | Since npm compat was added      |
| npm             | Runs all scripts by default                                        | —                               |

npm is the only one that still runs everything by default.

## Scope

This RFC covers the following lifecycle scripts when they are defined by dependencies (not the root project):

- `preinstall`
- `install`
- `postinstall`
- Auto-detected `binding.gyp` files (which trigger implicit `node-gyp rebuild`)
- `prepare` (for non-registry sources only: git dependencies, local file/link dependencies)

The following are out of scope:

- Scripts defined in the root project's `package.json` (these always run, as they are under the developer's direct control)
- Lifecycle scripts triggered by explicit user action (`test`, `start`, `stop`, `restart`, `publish`, etc.)
- Registry-level changes (2FA requirements, package signing, provenance attestations)
- Runtime code isolation or sandboxing

## Detailed Explanation

### Design overview

The design is modeled on pnpm v10's `allowBuilds` system, adapted for npm conventions:

1. A new `allowScripts` field in `package.json` declares which dependencies are permitted to run install scripts.
2. A new `npm approve-scripts` command provides an interactive workflow for managing the allowlist.
3. Dependency install scripts not covered by the allowlist are blocked, with a clear warning and remediation instructions.
4. A phased rollout allows the ecosystem to migrate gradually.

### The `allowScripts` field

A new top-level field in `package.json` maps package name patterns to `true` (allowed) or `false` (denied):

```json
{
  "allowScripts": {
    "canvas": true,
    "sharp": true,
    "core-js": false,
    "nx@21.6.4 || 21.6.5": true
  }
}
```

The field uses three values:

| Value   | Behavior                                          |
|---------|---------------------------------------------------|
| `true`  | Install scripts are permitted                     |
| `false` | Install scripts are blocked silently (no warning) |
| Absent  | Install scripts are blocked with a warning        |

This three-value design (allow / deny / unreviewed) matches pnpm's `allowBuilds` semantics. The distinction between `false` and absent is important: `false` means "I have reviewed this package and decided it does not need scripts," while absent means "this package has not been reviewed yet."

Package entries may include version constraints using the `@` separator:

```json
{
  "allowScripts": {
    "sharp": true,
    "nx@21.6.4 || 21.6.5": true,
    "sqlite3@5.1.7": true
  }
}
```

A name-only entry (e.g., `"sharp": true`) allows all versions. A versioned entry (e.g., `"nx@21.6.4 || 21.6.5": true`) restricts the allowance to specific versions. If both a name-only entry and a versioned entry exist for the same package, the versioned entry takes precedence for matching versions.

The `allowScripts` field is only read from the root project's `package.json` (or workspace root). `allowScripts` fields in dependency `package.json` files are ignored. This is a consumer-side policy, not a publisher declaration.

### Policy layering

Script permissions are resolved from multiple sources with the following precedence (highest to lowest):

1. CLI flags (`--allow-scripts`, `--no-scripts`, `--dangerously-allow-all-scripts`)
2. Project `package.json` (`allowScripts` field)
3. User `.npmrc` (`allow-scripts` setting)
4. Global `.npmrc` (`allow-scripts` setting)

The `.npmrc` setting provides a fallback for contexts without a project `package.json`, such as `npm install -g` and `npx`:

```ini
; ~/.npmrc
allow-scripts = canvas, sharp, sqlite3
```

### The `npm approve-scripts` command

A new interactive command scans the dependency tree, identifies packages with install scripts, and prompts the user to approve or deny each one:

```
$ npm approve-scripts
The following packages want to run install scripts:

? Select packages to approve (Press <space> to select, <a> to toggle all)
❯ ○ canvas (postinstall: node-gyp rebuild)
  ○ sharp (install: node install/libvips && ...)
  ○ sqlite3 (install: node-pre-gyp install ...)
  ○ core-js (postinstall: node -e "try{require('./postins...")
```

After selection, approved packages are written to `allowScripts` with `true` and denied packages with `false`. Newly approved packages are immediately rebuilt.

Non-interactive usage:

```sh
# Approve specific packages
npm approve-scripts canvas sharp

# Deny specific packages
npm approve-scripts !core-js

# Approve all pending
npm approve-scripts --all
```

### The `npm ignored-scripts` command

Lists all dependencies whose install scripts were blocked during the last install:

```
$ npm ignored-scripts
The following packages had their install scripts blocked:

  canvas (postinstall: node-gyp rebuild)
  sharp (install: node install/libvips && ...)

To approve them, run: npm approve-scripts
```

### Enforcement behavior

When a dependency has install scripts and is not in the `allowScripts` allowlist:

- The install continues by default. Scripts are skipped, and a warning is printed listing the blocked packages with a suggestion to run `npm approve-scripts`.
- In strict mode (`strict-script-builds=true` in `.npmrc`), the install fails with an error instead.
- The `--dangerously-allow-all-scripts` flag overrides the allowlist and runs all scripts.

The existing `--ignore-scripts` flag continues to work as before, disabling all scripts including root project scripts.

### Affected commands

The script policy is enforced by the following commands:

| Command                  | Behavior                                 |
|--------------------------|------------------------------------------|
| `npm install` / `npm ci` | Enforce `allowScripts` policy            |
| `npm rebuild`            | Enforce `allowScripts` policy            |
| `npm install -g`         | Enforce policy from user/global `.npmrc` |
| `npx` / `npm exec`       | Enforce policy from user/global `.npmrc` |
| `npm update`             | Enforce `allowScripts` policy            |

### Workspaces

In a workspace (monorepo) context:

- The root `package.json` `allowScripts` field is the single source of truth for the entire workspace.
- Individual workspace `package.json` files do not have their own `allowScripts` fields. All script permissions are managed at the root.
- This avoids ambiguity about merge semantics and ensures security policy is set in one place.

### Optional dependencies

If a package in `optionalDependencies` has install scripts that are blocked, it is treated as a failed optional dependency installation. This is consistent with existing behavior where optional dependencies that fail to build are silently skipped.

### Detecting unexpected script changes

When a `package-lock.json` is present, npm already records the full metadata for each resolved package. The implementation should additionally track which packages had install scripts at lock time. If a subsequent `npm install` resolves a package version whose scripts differ from what was recorded in the lock file (e.g., a previously script-free package now has a `postinstall`), npm should treat this as an unreviewed package and block the script with a warning, even if a name-only `allowScripts` entry exists for that package.

This provides a [Trust On First Use (TOFU)](https://en.wikipedia.org/wiki/Trust_on_first_use) model: the first install establishes a baseline, and changes to script presence trigger review. This directly addresses the attack pattern where a compromised maintainer adds a `postinstall` script to a patch release of a previously script-free package.

## Rationale and Alternatives

### Why not keep `--ignore-scripts` as-is?

The existing `--ignore-scripts` flag is all-or-nothing: it disables scripts for every package including the root project. This makes it impractical for projects that need some packages to build (e.g., `sharp` for image processing) while blocking scripts from the rest of the dependency tree. A per-package allowlist solves this.

### Why not use 2FA requirements instead?

Several commenters on the original RFC suggested requiring 2FA for all publishers as an alternative. 2FA reduces the risk of account takeover, but it does not address:

- Token theft from CI systems (automated publishing uses tokens, not 2FA)
- Insider threats (a legitimate maintainer can publish a malicious version)
- The fact that install scripts run code during installation, before any human reviews the published content

2FA and install script controls address different parts of the supply chain. They work well together but neither replaces the other.

### Why not a runtime sandbox?

Sandboxing install scripts (restricting file system or network access) is worth exploring separately, but it is a harder problem with more compatibility risk. An allowlist is simpler: if a package isn't on the list, its scripts don't run. Both approaches can coexist.

### Why not malware scanning?

Scanning packages for malicious code is useful but reactive: it depends on someone identifying the threat after publication. The allowlist approach works the other way around. Unknown or unreviewed packages simply cannot run install scripts, whether or not they have been scanned.

### Why a map instead of an array?

Bun uses a `trustedDependencies` array of package names. pnpm's `allowBuilds` uses a map. The map approach is better because:

- It supports explicit denial (`false`) vs. unreviewed (absent), enabling a clear audit trail.
- It accommodates version pinning as a key in the map entry.
- It is extensible to future per-package configuration if needed.

## Implementation

### npm CLI changes

The primary enforcement point is in the `@npmcli/run-script` and `@npmcli/arborist` packages:

1. `@npmcli/arborist`: During the `reify` step, before running lifecycle scripts for each dependency, check the resolved package name and version against the root project's `allowScripts` field. If the package is not allowed, skip its scripts and record it in a "blocked scripts" list.

2. `@npmcli/run-script`: Add an `allowed` check that consults the policy stack (CLI flags -> `package.json` -> `.npmrc`). When a script is blocked, emit a warning (or error in strict mode) with the package name, script name, and remediation command.

3. `npm approve-scripts` (new command): Reads the current `node_modules` tree (from `package-lock.json` or disk), identifies packages with install scripts not yet in `allowScripts`, and writes decisions to `package.json`. Triggers `npm rebuild` for newly approved packages. This command has two modes:

    - Non-interactive (works today): positional arguments (`npm approve-scripts canvas sharp`) and `--all` use only the existing `read` package and `proc-log` input primitives, which are already in the CLI.
    - Interactive multi-select (new dependency required): the arrow-key/checkbox UI shown in the design section would require adding a terminal prompt library such as `enquirer` or `@inquirer/prompts`. The npm CLI does not currently bundle anything capable of multi-select prompts; its `read` package only handles single-line text input. pnpm solved this by depending on `enquirer`. The interactive mode could ship in a follow-up if adding a new dependency is contentious.

4. `npm ignored-scripts` (new command): Reads the "blocked scripts" metadata (stored in `node_modules/.package-lock.json` or a similar location) and prints a summary.

### Configuration

New `.npmrc` settings:

| Setting                         | Type                 | Default | Description                                                      |
|---------------------------------|----------------------|---------|------------------------------------------------------------------|
| `allow-scripts`                 | Comma-separated list | (empty) | Packages allowed to run install scripts (for global/npx context) |
| `strict-script-builds`          | Boolean              | `false` | When `true`, blocked scripts cause install to fail               |
| `dangerously-allow-all-scripts` | Boolean              | `false` | When `true`, all scripts run (escape hatch)                      |

## Prior Art

### Package managers

- [pnpm v10](https://pnpm.io/settings#allowbuilds) (`allowBuilds` map in `pnpm-workspace.yaml`, `pnpm approve-builds` interactive CLI, `strictDepBuilds`, `dangerouslyAllowAllBuilds`). This is the primary model for this RFC. pnpm also ships related supply chain features: `minimumReleaseAge` (delay installing newly published versions), `trustPolicy` (fail on trust level downgrade), and `blockExoticSubdeps` (restrict transitive git/tarball sources).
- [Yarn Berry](https://yarnpkg.com/configuration/yarnrc#enableScripts) (`enableScripts: false` in `.yarnrc.yml`, per-package `dependenciesMeta.built` in `package.json`). Yarn was the first major package manager to default scripts off (v2, 2020). Its per-package control is in `package.json`, but there is no interactive approval command.
- [Bun](https://bun.sh/docs/install/lifecycle) (`trustedDependencies` array in `package.json`, ~400 hardcoded default-trusted packages, `bun pm trust` and `bun pm untrusted` CLI commands). Bun's default-trusted list reduces migration friction but creates a security surface: any compromise of a default-trusted package affects all Bun users.
- Deno: Blocks npm lifecycle scripts by default. Per-package opt-in via `deno install --allow-scripts=npm:sqlite3`. Also has an unstable `--minimum-dependency-age` flag.

### Community tools

- [@lavamoat/allow-scripts](https://www.npmjs.com/package/@lavamoat/allow-scripts): Manages an allowlist in `package.json`, runs via `npm install --ignore-scripts && npx allow-scripts`.
- [can-i-ignore-scripts](https://www.npmjs.com/package/can-i-ignore-scripts): Scans `node_modules` and categorizes packages by whether their install scripts can be safely skipped. Useful as a migration assessment tool.

### Related npm RFCs

- [RFC #861](https://github.com/npm/rfcs/pull/861): "Add option to require install script approval." Adds opt-in controls without changing defaults. This RFC supersedes #861 with a broader scope (default-deny).
- [RFC #92](https://github.com/npm/rfcs/pull/92): "Add staging workflow for CI and human interoperability." A publish-side security proposal (closed without implementation).

## Migration Plan

### Phase 1: Tooling and advisory warnings (next minor release)

- Ship `npm approve-scripts` and `npm ignored-scripts` commands.
- Recognize the `allowScripts` field in `package.json`.
- Print advisory warnings when dependency install scripts run that are not covered by an `allowScripts` field.
- No change in default behavior: scripts still run.

### Phase 2: Default-deny (next major release)

- Dependency install scripts are blocked by default.
- Scripts for packages listed in `allowScripts` with `true` still run.
- Blocked scripts produce a warning with remediation instructions.
- `strict-script-builds=true` available for CI environments that want hard failures.
- `--dangerously-allow-all-scripts` available as an escape hatch.

### Phase 3: Ecosystem stabilization

- Monitor adoption, gather feedback, iterate on `npm approve-scripts`.
- Evaluate related features (e.g., `minimum-release-age`, `trust-policy`) as separate RFCs.

## Unresolved Questions and Bikeshedding

1. Field name: this RFC proposes `allowScripts`. Alternatives include `allowBuilds` (matching pnpm's naming) or `trustedDependencies` (matching Bun). `allowScripts` is the most self-descriptive for npm's context, where the feature controls lifecycle *scripts* rather than *builds* in the broader sense.

2. Script content preview: should `npm approve-scripts` display the actual script contents (e.g., `"postinstall": "node-gyp rebuild"`) to help users make informed decisions? pnpm shows package names only; Bun shows script names. Showing full script content adds security value but may be noisy for long scripts.

3. Provenance integration: should packages published with [npm provenance](https://docs.npmjs.com/generating-provenance-statements) attestations receive any preferential treatment in the script policy? For example, a future `trust-policy` setting could allow scripts only from packages with verified provenance. This is deferred to a follow-up RFC.

4. Version pinning default: this RFC allows both name-only (`"sharp": true`) and version-pinned (`"sharp@0.33.2": true`) entries. Should the default output of `npm approve-scripts` pin to the currently installed version, or use name-only? Name-only is more convenient; version-pinned is more secure.

5. Remaining native addon packages: the ecosystem has largely shifted to prebuilt binaries, but some packages still require `node-gyp` at install time. Updated data on how many of the top-downloaded packages still use install scripts would strengthen the migration plan. If the number is small enough, the default-deny change is justified without a new npm-side mechanism for native addon distribution.

6. `.npmrc` expressiveness: the `.npmrc` format (comma-separated list of package names) cannot express the full tri-state + version-pinning model available in `package.json`. This is acceptable for the global/npx use case (where fine-grained control matters less), but the limitation should be documented.
