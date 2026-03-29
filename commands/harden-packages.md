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

### 4. Check for package cooldown enforcement (MEDIUM)

Check whether the project has any mechanism to avoid installing freshly-published packages — the highest-risk window for supply chain attacks. Look for:

- `renovate.json` or `renovate.json5` containing `minimumReleaseAge`
- SafeChain installed (`which safe-chain`)
- Socket Firewall configuration
- CI steps that check package age

If **none are found**, flag this in the summary and recommend an approach based on the project's setup:

- **If the project already uses Renovate**: suggest adding `"minimumReleaseAge": "3 days"` to `renovate.json` — this is the lowest-friction option and covers all ecosystems.
- **If the project uses Dependabot**: note that Dependabot doesn't have an equivalent setting. Suggest switching to Renovate or adding SafeChain/Socket Firewall alongside it.
- **If npm/pnpm/yarn or pip/uv/poetry**: suggest [Aikido SafeChain](https://github.com/AikidoSec/safe-chain) — it enforces a 48hr cooldown on new npm packages and blocks known-malicious packages at install time for JS and Python.
- **For all other ecosystems** (Go, Rust, Ruby, JVM): suggest Renovate `minimumReleaseAge` as the primary option.

Explain **why** this matters: lockfiles pin versions but don't enforce age. When a developer runs `npm install new-package` or updates a dependency, the freshly-published version gets locked in immediately. A cooldown period gives the community time to detect compromised releases before they reach your lockfile.

Do NOT auto-configure this — it depends on the team's workflow. Flag it and recommend.

See `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/package-configs.md` for configuration examples.

### 5. Add registry scoping (MEDIUM)

If the project uses scoped packages (e.g., `@yourorg/`), check for registry scoping in `.npmrc` or equivalent. If missing, add a commented-out template and explain how to configure it to prevent dependency confusion attacks.

### 6. Run available scanners

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
- **For JavaScript/Python projects**: install [Aikido SafeChain](https://github.com/AikidoSec/safe-chain) to block malicious packages at install time (`curl -fsSL https://github.com/AikidoSec/safe-chain/releases/latest/download/install-safe-chain.sh | sh`)
```
