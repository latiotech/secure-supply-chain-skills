---
description: Run a full supply chain security audit across all domains
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, npm:*, pnpm:*, yarn:*, pip:*, docker:*, terraform:*, tofu:*, gh:*, which:*, betterleaks:*, zizmor:*, hadolint:*, checkov:*, modelscan:*)
---

Run a comprehensive supply chain security audit across the entire project. This is the "start here" command - it detects what's in the repo, checks everything, and **takes immediate action on the highest-priority items**.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/checklist.md` for the full checklist.

## Phase 1: Discovery

Scan the project to determine which supply chain domains are relevant:

1. **Package managers** - look for package.json, requirements.txt, go.mod, Cargo.toml, Gemfile, pom.xml, build.gradle
2. **Container images** - look for Dockerfile*, docker-compose*, .dockerignore
3. **GitHub Actions** - look for .github/workflows/*.yml
4. **Infrastructure-as-Code** - look for *.tf, terragrunt.hcl, *.tfvars
5. **AI/ML models** - look for *.pkl, *.pt, *.pth, *.onnx, *.safetensors, or references to huggingface/transformers in dependencies
6. **IDE extensions** - look for .vscode/extensions.json, .devcontainer/devcontainer.json
7. **Credentials** - look for .env files, hardcoded secrets, .npmrc with auth tokens, .pypirc, private keys

Report what was found.

## Phase 1.5: Scanner Prerequisites

Based on which domains were detected, check for the recommended scanner for each domain. Only suggest scanners for domains that are actually present. Use `which` to check availability. Suggest **one tool per domain** — no overlapping suggestions.

| Domain | Scanner | Check | Install |
|--------|---------|-------|---------|
| GitHub Actions | zizmor | `which zizmor` | `brew install zizmor` or `cargo install zizmor` |
| Credentials | betterleaks | `which betterleaks` | `brew install betterleaks` or `cargo install betterleaks` |
| Containers | hadolint | `which hadolint` | `brew install hadolint` |
| IaC | checkov | `which checkov` | `pip install checkov` |
| AI/ML (unsafe model files only) | modelscan | `which modelscan` | `pip install modelscan` |

For each detected domain, check if its scanner is installed. Collect all missing scanners into a single recommendation block:

> **Recommended scanners for your project:**
>
> The following scanners would improve this audit:
>
> ```bash
> brew install zizmor        # GitHub Actions security
> brew install betterleaks   # Secret scanning
> ```
>
> Scanner-driven results are significantly more accurate than pattern-based analysis. Would you like me to install any of these before continuing?

Only list scanners that are (a) relevant to detected domains and (b) not already installed. If all relevant scanners are installed, skip this section entirely.

**Wait for the user to respond before proceeding to Phase 2.** If the user wants to install scanners, install them first. If they decline or want to move on, continue with pattern-based analysis for domains without scanners.

## Phase 2: Quick Audit

For each detected domain, run the highest-priority checks:

### Packages
- Count pinned vs unpinned dependencies
- Check if install scripts are disabled
- Verify lockfile is committed
- Check for package cooldown enforcement — look for Renovate `minimumReleaseAge`, SafeChain installation, Socket Firewall config, or equivalent. If none found, flag as **MEDIUM**: the lockfile pins versions but doesn't prevent a recently-published malicious version from being locked in when someone runs `npm install` / `pip install`

### Containers
- If hadolint is installed, run `hadolint` against each Dockerfile and include findings
- Check if base images are pinned by digest
- Check for non-root USER directives
- Scan for hardcoded secrets

### GitHub Actions
- If zizmor is installed, run `zizmor .github/workflows/` and include findings
- Count SHA-pinned vs tag-pinned Actions
- Check for explicit permissions
- Flag any `pull_request_target` triggers
- Check CODEOWNERS coverage

### IaC
- If checkov is installed, run `checkov -d . --framework terraform --compact` and include findings
- Check module and provider pinning
- Verify lockfile is committed
- Flag `local-exec` provisioners

### AI/ML
- If modelscan is installed and unsafe model files (`.pkl`, `.pt`, `.pth`, `.joblib`) exist, run `modelscan --path {file}` against each
- Flag pickle/torch.load calls without safety flags
- Check for committed model files

### IDE Extensions
- Check recommended extensions for verified publishers
- Scan for secrets in workspace settings

### Credentials
- If betterleaks is installed, run `betterleaks --path .` and include findings
- Scan for hardcoded secrets and API keys in source code (pattern-based fallback/supplement)
- Check if .env files are gitignored
- Check git history for previously committed secrets
- Flag package manager configs with embedded auth tokens

## Phase 3: Immediate Fixes

**Take action on CRITICAL items automatically:**

- Pin unpinned dependency versions to their current resolved versions
- Pin GitHub Actions to commit SHAs (resolve the actual SHA for each tag)
- Add `ignore-scripts=true` to `.npmrc` if missing
- Add `weights_only=True` to `torch.load()` calls
- Set explicit `permissions:` on workflows that lack them
- Add `save-exact=true` to `.npmrc` if missing
- Add missing secret file patterns to `.gitignore`

Explain each change as it's made.

## Phase 4: Report

Generate a summary report:

```
## Supply Chain Security Audit

**Project**: [name]
**Date**: [date]
**Domains detected**: [list]
**Scanners used**: [list installed scanners that were run, or "none — pattern-based only"]

### Scorecard

| Domain | Status | Critical Issues | Fixed | Remaining |
|--------|--------|----------------|-------|-----------|
| Packages | ⚠️ | 3 unpinned deps | 3 pinned | 0 |
| Actions | 🔴 | 5 unpinned, no perms | 5 pinned, perms added | OIDC needed |
| ... | ... | ... | ... | ... |

### Changes Made
[List each change with file and description]

### Requires Manual Attention
[Items that couldn't be auto-fixed, numbered for reference]

After presenting this section, ask the user:

> Would you like me to investigate and fix any of these? Give me the number(s) or describe which ones, and I'll dig in.

If the user picks items, investigate each one in detail:
- Read the relevant files and surrounding context
- Explain the specific risk and how an attacker could exploit it
- Propose a concrete fix with the tradeoffs involved
- Apply the fix if the user agrees, or explain the manual steps if it requires external action (GitHub settings, cloud config, etc.)

Continue offering to address remaining items until the user is satisfied or wants to move on.

### Ongoing Protection

**Package cooldown** — newly published package versions are highest-risk. Recommend one of these approaches based on the project's ecosystem and setup:

| Approach | Ecosystems | How It Works |
|----------|-----------|--------------|
| [Aikido SafeChain](https://github.com/AikidoSec/safe-chain) | npm, pip | Local install-time interceptor; blocks malicious packages and enforces 48hr cooldown on new npm versions. Free, zero config. |
| [Renovate](https://docs.renovatebot.com/configuration-options/#minimumreleaseage) `minimumReleaseAge` | All | Delays dependency update PRs by a configurable period (e.g., 3 days). Free, works for every ecosystem Renovate supports. |
| [Socket Firewall](https://docs.socket.dev/docs/socket-firewall-overview) | npm, pip, cargo | Registry proxy that intercepts installs and blocks suspicious packages. Freemium. |
| [Sonatype Repository Firewall](https://www.sonatype.com/products/sonatype-repository-firewall) | All (via proxy) | Quarantines suspicious components at the artifact repository level. Paid, enterprise. |

If the project already uses Renovate or Dependabot, the lowest-friction option is adding `minimumReleaseAge`. For JS/Python projects without a dependency updater, SafeChain is the easiest standalone option.

### Maintenance
- `/update-pins` - Check all pinned deps/actions/images for newer versions and update them
- `/minimize` - Remove unused dependencies and convert Dockerfiles to multi-stage builds

### Domain-Specific Hardening
- `/harden-credentials` - Deep secret scanning, pre-commit hooks, credential hygiene

### Advanced Hardening (Run Separately)
- `/setup-oidc` - Replace long-lived cloud credentials with OIDC tokens
- `/setup-image-signing` - Sign container images with Cosign/Sigstore
- `/setup-tag-rulesets` - Protect release tags from force-push attacks
- `/setup-admission-control` - Enforce signed-only images in Kubernetes
- `/setup-sbom` - Generate SBOMs and provenance attestations
- `/setup-commit-signing` - Set up commit and tag signing (SSH, GPG, or gitsign)
- `/setup-runner-monitoring` - Add runtime detection to CI runners
```

Save the report to `supply-chain-audit.md` in the project root.
