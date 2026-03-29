---
description: Run a full supply chain security audit across all domains
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, npm:*, pnpm:*, yarn:*, pip:*, docker:*, terraform:*, tofu:*, gh:*)
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

## Phase 2: Quick Audit

For each detected domain, run the highest-priority checks:

### Packages
- Count pinned vs unpinned dependencies
- Check if install scripts are disabled
- Verify lockfile is committed

### Containers
- Check if base images are pinned by digest
- Check for non-root USER directives
- Scan for hardcoded secrets

### GitHub Actions
- Count SHA-pinned vs tag-pinned Actions
- Check for explicit permissions
- Flag any `pull_request_target` triggers
- Check CODEOWNERS coverage

### IaC
- Check module and provider pinning
- Verify lockfile is committed
- Flag `local-exec` provisioners

### AI/ML
- Flag pickle/torch.load calls without safety flags
- Check for committed model files

### IDE Extensions
- Check recommended extensions for verified publishers
- Scan for secrets in workspace settings

### Credentials
- Scan for hardcoded secrets and API keys in source code
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

### Scorecard

| Domain | Status | Critical Issues | Fixed | Remaining |
|--------|--------|----------------|-------|-----------|
| Packages | ⚠️ | 3 unpinned deps | 3 pinned | 0 |
| Actions | 🔴 | 5 unpinned, no perms | 5 pinned, perms added | OIDC needed |
| ... | ... | ... | ... | ... |

### Changes Made
[List each change with file and description]

### Requires Manual Attention
[Items that couldn't be auto-fixed]

### Domain-Specific Hardening
- `/harden-credentials` - Deep secret scanning, pre-commit hooks, credential hygiene

### Advanced Hardening (Run Separately)
- `/setup-oidc` - Replace long-lived cloud credentials with OIDC tokens
- `/setup-image-signing` - Sign container images with Cosign/Sigstore
- `/setup-tag-rulesets` - Protect release tags from force-push attacks
- `/setup-admission-control` - Enforce signed-only images in Kubernetes
- `/setup-sbom` - Generate SBOMs and provenance attestations
- `/setup-runner-monitoring` - Add runtime detection to CI runners
```

Save the report to `supply-chain-audit.md` in the project root.
