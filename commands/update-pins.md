---
description: Check pinned dependencies, actions, and images for newer versions and update their pins
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, npm:*, pnpm:*, yarn:*, pip:*, docker:*, terraform:*, tofu:*, curl:*, which:*, jq:*)
argument-hint: [domain: packages|actions|containers|iac|all]
---

Check all pinned dependencies for available updates, assess migration difficulty, and update pins to the latest versions. **This command takes action by default** — it resolves new SHAs and digests, flags breaking changes, and updates pins. Changes are explained as they are made.

If $1 is provided, limit to that domain. Otherwise, check all detected domains.

## Phase 1: Inventory Current Pins

Scan the project to build an inventory of everything that's currently pinned. For each pinned item, record:
- **What**: the dependency name, current version/SHA/digest
- **Where**: file and line number
- **Type**: package version, Action SHA, container digest, Terraform module/provider version

### Packages
Find all dependency files (`package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `Gemfile`, `pom.xml`, `build.gradle`) and list each dependency with its pinned version.

### GitHub Actions
Find all `.github/workflows/*.yml` and `.github/actions/*/action.yml` files. For each `uses:` directive pinned to a SHA, extract:
- The owner/repo
- The current SHA
- The version comment (e.g., `# v4.1.1`) if present

### Containers
Find all Dockerfiles. For each `FROM` directive pinned to a digest (`@sha256:...`), extract:
- The image name and tag
- The current digest

### IaC
Find all `*.tf` files. For each module or provider with a pinned version (`= x.y.z`), extract:
- The source (registry path or git URL)
- The current version

## Phase 2: Look Up Latest Versions

For each pinned item, look up the latest available version. **Use deterministic resolution methods** — the same methods from the hardening commands.

### Packages

Use the appropriate registry API for each ecosystem:

- **npm**: `npm view {package} version` for latest, `npm view {package} versions --json` to see all versions
- **pip/PyPI**: `curl -s https://pypi.org/pypi/{package}/json | jq -r '.info.version'`
- **Go**: `curl -s https://proxy.golang.org/{module}/@latest | jq -r '.Version'`
- **Cargo**: `curl -s https://crates.io/api/v1/crates/{crate} | jq -r '.crate.newest_version'`
- **Ruby**: `curl -s https://rubygems.org/api/v1/gems/{gem}.json | jq -r '.version'`

If a lookup fails, skip that dependency and note it as unresolvable.

### GitHub Actions

For each pinned action, resolve the latest release tag and its commit SHA:

1. Get the latest release tag:
   ```bash
   gh api repos/{owner}/{repo}/releases/latest --jq '.tag_name'
   ```
   If no releases, use tags: `git ls-remote --tags --sort=-v:refname https://github.com/{owner}/{repo}.git | head -1`

2. Resolve the commit SHA for that tag (dereference annotated tags):
   ```bash
   git ls-remote https://github.com/{owner}/{repo}.git "refs/tags/{tag}^{}"
   ```
   If empty (lightweight tag), fall back to `refs/tags/{tag}`.

### Containers

For each pinned image, resolve the latest digest for the same tag:

```bash
docker buildx imagetools inspect {image}:{tag} --format '{{json .Manifest}}' | jq -r '.digest'
```

If the tag itself has moved (e.g., `node:18-alpine` now points to a newer build), report the new digest.

Also check if a newer tag series exists (e.g., `node:20-alpine` when pinned to `node:18-alpine`). Report this separately as a **tag upgrade** — distinct from a digest refresh.

### IaC

For each Terraform module/provider:

- **Registry modules**: `curl -s https://registry.terraform.io/v1/modules/{namespace}/{name}/{provider} | jq -r '.version'`
- **Registry providers**: `curl -s https://registry.terraform.io/v1/providers/{namespace}/{name} | jq -r '.version'`
- **Git-sourced modules**: `git ls-remote --tags --sort=-v:refname {url} | head -1` then dereference with `^{}`

## Phase 3: Assess Migration Difficulty

For each item where the latest version differs from the current pin, classify the update:

### Version bump classification

| Classification | Criteria | Action |
|---------------|----------|--------|
| **Patch** (e.g., 4.1.1 → 4.1.2) | Patch version change only | Update automatically |
| **Minor** (e.g., 4.1.1 → 4.2.0) | Minor version change | Update automatically, note in summary |
| **Major** (e.g., 4.1.1 → 5.0.0) | Major version change | **Do NOT auto-update.** Flag with changelog link and list known breaking changes |
| **Digest refresh** (same tag, new digest) | Container tag points to newer build | Update automatically — same tag means same compatibility contract |
| **Tag upgrade** (e.g., node:18 → node:20) | Newer tag series available | **Do NOT auto-update.** Flag as optional upgrade with migration notes |

### How to assess breaking changes for major bumps

For GitHub Actions:
```bash
gh api repos/{owner}/{repo}/releases --jq '.[] | select(.tag_name | startswith("v{major}")) | {tag_name, body}' | head -50
```
Scan release notes for "BREAKING", "breaking change", "migration", "removed", "deprecated".

For npm packages, check for peer dependency changes:
```bash
npm view {package}@latest peerDependencies --json
```

For all: include a direct link to the changelog or release notes so the user can review.

## Phase 4: Apply Updates

### Auto-apply (patch, minor, digest refresh)

For each update classified as patch, minor, or digest refresh:
- Update the pin in place
- For Actions: update the SHA and the version comment
- For containers: update the digest
- For packages: update the version number
- For IaC: update the version constraint
- Explain each change

### Flag only (major, tag upgrade)

For each major version bump or tag upgrade, do NOT make changes. Instead, collect them for the summary with:
- Current version → latest version
- Link to changelog/release notes
- Known breaking changes (if found in release notes)
- Suggested migration steps (if obvious from the changelog)

## Output

After all updates, provide a summary:

```
## Pin Update Summary

**Scanned**: N pinned dependencies across M files
**Updated**: X items | **Flagged**: Y items needing manual review

### Updates Applied

| Item | File | Previous | Updated To | Type |
|------|------|----------|-----------|------|
| actions/checkout | ci.yml:12 | abc123 (v4.1.1) | def456 (v4.2.0) | Minor |
| node | Dockerfile:1 | sha256:aaa... | sha256:bbb... | Digest refresh |
| lodash | package.json:8 | 4.17.20 | 4.17.21 | Patch |

### Major Updates Available (Manual Review Required)

| Item | File | Current | Latest | Breaking Changes |
|------|------|---------|--------|-----------------|
| actions/upload-artifact | ci.yml:34 | v3.1.3 | v4.3.1 | [Changelog](link) - switched to artifact client |
| express | package.json:5 | 4.18.2 | 5.0.0 | [Release notes](link) - path matching changes |

### Already Up To Date
- N items are already on the latest version

### Recommended Next Steps
- Review major version updates above and apply when ready
- Run `/audit-supply-chain` to verify no new issues were introduced
- Run your test suite to confirm nothing broke
```
