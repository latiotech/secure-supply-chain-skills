---
description: Pin dependency versions, disable install scripts, and secure registry configs
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm:*, pnpm:*, yarn:*, pip:*, pip-compile:*, uv:*, go:*, cargo:*, bundle:*, git:*, docker:*)
argument-hint: [package-manager]
---

Harden the project's package manager configuration for supply chain security. **This command takes action by default** - it pins versions, disables scripts, and secures configs. Changes are explained as they are made.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/package-configs.md` for language-specific configurations.

## Detection

First, detect which package managers are in use by looking for these files:

- `package.json` / `package-lock.json` / `pnpm-lock.yaml` / `yarn.lock` → npm/pnpm/yarn
- `requirements.txt` / `requirements.in` / `pyproject.toml` / `uv.lock` → pip/uv
- `go.mod` / `go.sum` → Go
- `Cargo.toml` / `Cargo.lock` → Rust
- `Gemfile` / `Gemfile.lock` → Ruby
- `pom.xml` / `build.gradle` → Java

If $1 is provided, focus on that package manager only.

## Action Plan

For each detected package manager, work through these in order. **Make each change, explain what it does and why, then move to the next.**

### 1. Pin all dependency versions (CRITICAL)

Scan dependency files for unpinned versions (ranges like `^`, `~`, `>=`, `*`). For each unpinned dependency:
- Resolve the current version from the lockfile
- If no lockfile exists, run the package manager's install/resolve command to generate one, then read versions from it
- If neither is possible, add a `# TODO: pin to exact version - run [lock command] and read the resolved version` comment. **Never guess a version number.**
- Replace the range with the exact pinned version
- Show what was changed and why (floating ranges allow silent upgrades to compromised versions)

For npm/pnpm/yarn: also add `save-exact=true` to `.npmrc` so future installs pin automatically.
For pip: generate pinned requirements with hashes using `pip-compile --generate-hashes` if pip-tools is available.
For Go: run `go mod tidy && go mod verify` to ensure checksums are valid.

### 2. Disable install scripts (HIGH)

Add the appropriate config to block pre/post-install script execution:
- npm/pnpm: add `ignore-scripts=true` to `.npmrc` (create the file if it doesn't exist)
- yarn: add `enableScripts: false` to `.yarnrc.yml`
- pip: add `require-hashes = true` to pip.conf if not already present

If specific packages are known to need install scripts (e.g., esbuild, sharp, bcrypt), add them to an allowlist in the appropriate config format.

### 3. Ensure lockfile is committed (HIGH)

Check if the lockfile exists and is not in `.gitignore`. If the lockfile is gitignored:
- Remove it from `.gitignore`
- Explain that lockfiles must be committed to ensure reproducible builds

### 4. Add registry scoping (MEDIUM)

If the project uses scoped packages (e.g., `@yourorg/`), check for registry scoping in `.npmrc` or equivalent. If missing, add a commented-out template and explain how to configure it to prevent dependency confusion attacks.

### 5. Run available scanners

If any of these tools are installed, run them and report results:
- `npm audit` / `pnpm audit` / `yarn audit`
- `cargo audit` (for Rust)
- `go mod verify` (for Go)
- `bundle audit` (for Ruby)

## Output

After making all changes, provide a summary:

```
## Package Hardening Summary

### Changes Made
- [x] Pinned N dependencies to exact versions in [file]
- [x] Added ignore-scripts=true to .npmrc
- [x] ...

### Manual Steps Needed
- [ ] Configure registry scoping for @yourorg packages
- [ ] Review allowScripts list for packages that need install scripts

### Recommended Next Steps
- Run `/setup-sbom` to generate a Software Bill of Materials
- Set up Dependabot or Renovate to keep pinned versions current
```
