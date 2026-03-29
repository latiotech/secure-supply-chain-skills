---
description: Pin Terraform modules, check state security, and flag dangerous provisioners
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(terraform:*, tofu:*, checkov:*, git:*, curl:*, which:*, pip:*, brew:*)
argument-hint: [directory]
---

Audit and harden Infrastructure-as-Code configurations for supply chain security. **This command takes action by default** - it pins module versions, generates lockfiles, and flags dangerous provisioners. Changes are explained as they are made.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/iac-configs.md` for detailed configurations.

## Detection

Find IaC files in the project (or $1 if a directory is provided):
- `*.tf` / `*.tf.json` → Terraform / OpenTofu
- `.terraform.lock.hcl` → Terraform lockfile
- `terragrunt.hcl` → Terragrunt

## Prerequisites

IaC files were found. Check if `checkov` is installed (`which checkov`).

If **not installed**, tell the user:

> **Recommended: install checkov** — a static analysis scanner for Terraform, CloudFormation, and Kubernetes that catches misconfigurations, insecure defaults, and compliance violations.
>
> ```bash
> pip install checkov
> # or with brew
> brew install checkov
> ```
>
> Once installed, re-run this command for scanner-driven results. Continuing with pattern-based analysis for now.

## Actions to Take

Work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Run Checkov (CRITICAL — if installed)

If checkov is installed, run it against the IaC directory:

```bash
checkov -d . --framework terraform --compact
```

Parse and report findings grouped by severity. Use Checkov's output to prioritize the fixes below — address scanner findings first. If Checkov flags an issue that a later step would also catch (e.g., unpinned modules, missing encryption), fix it here and note it as "confirmed by Checkov" when you reach that step.

### 2. Pin all modules and providers (CRITICAL)

Scan all `module` blocks for missing or loose `version` constraints (e.g., `>= 3.0`, no version at all).

For each unpinned module:
- If the current resolved version can be determined from `.terraform.lock.hcl` or state, pin to that exact version
- Otherwise, query the Terraform registry using: `curl -s https://registry.terraform.io/v1/modules/{namespace}/{name}/{provider} | jq .version`
- If the registry query fails or the module is from a non-registry source, add a `# TODO: pin to exact version - check registry or lockfile` comment. **Never fabricate a version number.**
- Change `>= 3.0` → `= 3.x.y` (exact pin)
- Explain: floating module versions can silently pull in compromised code

Do the same for `required_providers` blocks - pin each provider to an exact version.

### 3. Generate or verify lockfile (HIGH)

Check if `.terraform.lock.hcl` exists and is committed:
- If missing and terraform/tofu CLI is available, run `terraform providers lock -platform=linux_amd64 -platform=darwin_amd64 -platform=darwin_arm64`
- If missing and CLI is unavailable, add a comment and note the command to run
- If it exists but is in `.gitignore`, remove it from `.gitignore`
- Explain: the lockfile contains provider checksums; without it, a compromised registry can serve different binaries

### 4. Flag and restrict dangerous provisioners (HIGH)

Search for `local-exec` and `external` provisioners. For each:
- Add a prominent comment: `# WARNING: local-exec runs arbitrary commands - supply chain risk`
- If the provisioner runs a remote script (e.g., `curl | bash`), flag as CRITICAL
- Suggest alternatives (e.g., `terraform-provider-shell` with explicit allowlists, or moving logic to a proper module)
- Do NOT auto-remove provisioners - they may be load-bearing; flag and explain the risk

### 5. Check state security (MEDIUM)

Check backend configurations:
- If using S3: verify `encrypt = true` and `dynamodb_table` for locking are present. Add them if missing.
- If using GCS/Azure: verify encryption settings
- If state files (`*.tfstate`) are in git or not in `.gitignore`, add them to `.gitignore`
- Explain: state files contain secrets in plaintext and must be encrypted and access-controlled

### 6. Verify module sources (MEDIUM)

Check that modules come from trusted sources:
- Official Terraform registry (`registry.terraform.io`) - OK
- Org's private registry - OK
- Git sources with pinned refs (`?ref=v1.2.3`) - OK
- Git sources without pinned refs - resolve the current default branch HEAD using `git ls-remote {url} HEAD` and pin with `?ref={sha}`. If resolution fails, add a `# TODO: pin to commit SHA - run: git ls-remote {url} HEAD` comment. **Never fabricate a ref.**
- Git sources pinned to a tag (`?ref=v1.2.3`) - verify the ref resolves to a commit SHA, not a tag object. Use `git ls-remote {url} "refs/tags/v1.2.3^{}"` to dereference annotated tags. If you're pinning a module to a tag ref, prefer pinning to the commit SHA directly (`?ref={commit-sha}`) for immutability — tags can be force-pushed.

## Output

After making all changes, provide a summary:

```
## IaC Hardening Summary

### Changes Made
- [x] Pinned N modules to exact versions
- [x] Pinned N providers to exact versions
- [x] Generated/verified .terraform.lock.hcl
- [x] Added state files to .gitignore
- [x] Added encryption to backend config

### Requires Manual Attention
- [ ] local-exec provisioner in [file:line] - review and consider alternatives
- [ ] State backend missing locking - add DynamoDB table
- [ ] Module from unpinned git source in [file]

### Recommended Next Steps
- Set up OPA/Sentinel policies to enforce guardrails
- Configure drift detection with `terraform plan -detailed-exitcode` in CI
- Consider running IaC applies through Atlantis or Terraform Cloud
```
