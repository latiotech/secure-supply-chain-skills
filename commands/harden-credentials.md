---
description: Scan for leaked secrets, set up pre-commit hooks, and harden credential hygiene
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, betterleaks:*, pip:*, npx:*, curl:*, which:*, brew:*, cargo:*)
argument-hint: [directory]
---

Audit and harden credential hygiene across the project. **This command takes action by default** - it scans for leaked secrets, adds protective gitignore rules, sets up pre-commit hooks, and fixes credential anti-patterns. Changes are explained as they are made.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/credentials-configs.md` for detailed configurations and context.

## Detection

Find credential-related files and patterns in the project (or $1 if a directory is provided):

- `.env`, `.env.*` files (should never be committed)
- `.npmrc`, `.pypirc`, `.docker/config.json` with embedded auth tokens
- `*.pem`, `*.key`, `*.p12`, `*.pfx` files (private keys)
- `terraform.tfvars`, `*.auto.tfvars` with sensitive values
- Hardcoded API keys, tokens, passwords, and connection strings in source code
- Git history containing previously committed secrets

If no credential-related findings are detected, report that and exit.

## Prerequisites

Credential-related files or patterns were found. Check if `betterleaks` is installed (`which betterleaks`).

If **not installed**, tell the user:

> **Recommended: install betterleaks** — a fast, low-false-positive secret scanner that catches leaked credentials, API keys, and tokens across your codebase and git history.
>
> ```bash
> # macOS
> brew install betterleaks
> # or from source
> cargo install betterleaks
> ```
>
> Once installed, re-run this command for scanner-driven results. Continuing with pattern-based analysis for now.

## Actions to Take

Work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Run Betterleaks secret scan (CRITICAL — if installed)

If betterleaks is installed, run it:

```bash
betterleaks --path .
```

Parse and report all findings grouped by severity. For each leaked secret:
- Show the file, line number, and rule that matched (redact the actual secret value)
- Classify: test fixture, example/placeholder, or likely real credential
- If it's a likely real credential, flag as **CRITICAL** and recommend immediate rotation

Use Betterleaks output to prioritize the remaining steps. If Betterleaks already identified a secret that step 2 would catch, note it as "confirmed by Betterleaks" and skip the duplicate pattern check for that finding.

If betterleaks is not installed, rely entirely on the pattern-based scanning in step 2.

### 2. Pattern-based secret scanning (CRITICAL — fallback or supplement)

Regardless of whether betterleaks ran, scan for high-confidence patterns that tools sometimes miss:

- AWS keys: `AKIA[0-9A-Z]{16}`
- GitHub tokens: `ghp_[a-zA-Z0-9]{36}`, `gho_[a-zA-Z0-9]{36}`, `github_pat_[a-zA-Z0-9_]{80,}`
- Generic API keys: `sk-[a-zA-Z0-9]{20,}`, `sk_live_[a-zA-Z0-9]+`
- Private keys: `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`
- Connection strings with embedded passwords: `postgresql://.*:.*@`, `mongodb://.*:.*@`, `mysql://.*:.*@`
- Hardcoded passwords: `password\s*=\s*['"][^'"]{8,}['"]` (excluding obvious test values like "password123", "changeme", "test")

For each finding:
- Flag severity based on type (private keys and cloud credentials are CRITICAL; API keys are HIGH; generic passwords are MEDIUM)
- Redact the actual value in output
- Recommend the specific rotation steps

### 3. Harden .gitignore for secrets (CRITICAL)

Check `.gitignore` and add missing entries for sensitive file patterns. Add a clearly marked secrets section if these patterns are missing:

```
# Secrets and credentials
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.pfx
.npmrc
.pypirc
terraform.tfvars
*.auto.tfvars
.docker/config.json
```

Do NOT add entries that would conflict with existing rules. Preserve the existing file structure.

### 4. Scan git history for leaked secrets (HIGH)

Check if secrets were ever committed to git history:

```bash
git log --diff-filter=A --name-only --pretty=format: -- '*.env' '*.pem' '*.key' '*.p12' '*.pfx' '*credentials*' '*.tfvars' '.npmrc' '.pypirc'
```

If betterleaks supports history scanning, use it:
```bash
betterleaks --path . --scan-history
```

For each file found in history:
- Flag as **HIGH** - the secret is still in git history even if deleted from the working tree
- Recommend: rotate the credential immediately, then use `git-filter-repo` or BFG Repo Cleaner to remove from history
- Warn: all collaborators must re-clone after history rewriting

### 5. Suggest pre-commit hook for ongoing protection (MEDIUM)

If a `.pre-commit-config.yaml` already exists and doesn't include a secret scanning hook, **suggest** adding betterleaks:

```yaml
  - repo: https://github.com/betterleaks/betterleaks
    rev: v0.1.0  # TODO: check latest version
    hooks:
      - id: betterleaks
```

If no pre-commit config exists, mention that a betterleaks pre-commit hook is an option for catching secrets before they're committed — but do not create one automatically. The scan in step 1 is the primary defense.

### 6. Fix .env.example files (MEDIUM)

If `.env` files exist but no `.env.example`:
- Create `.env.example` with the same keys but placeholder values
- Ensure real `.env` files are gitignored

If `.env.example` exists, verify it doesn't contain real secret values (check for high-entropy strings).

### 7. Check package manager auth files (MEDIUM)

Scan for auth tokens in package manager configs:
- `.npmrc` with `_authToken=` or `_auth=` - these are long-lived tokens
- `.yarnrc.yml` with `npmAuthToken`
- `.pypirc` with `password =`

For each:
- If the file is tracked by git, flag as **HIGH**
- If the file is not in `.gitignore`, add it
- Recommend using credential helpers or environment variable injection instead of hardcoded tokens

## Output

After making all changes, provide a summary:

```
## Credential Hardening Summary

### Changes Made
- [x] Scanned codebase with betterleaks - found N secrets
- [x] Added N entries to .gitignore for sensitive file patterns
- [x] Created .env.example with placeholder values
- [x] Suggested betterleaks pre-commit hook (if .pre-commit-config.yaml exists)
- [x] Flagged N auth tokens in package manager configs

### Requires Manual Attention
- [ ] Rotate [credential] found in [file] - it may be compromised
- [ ] Remove secrets from git history using git-filter-repo
- [ ] Rotate [N] credentials found in git history
- [ ] Replace hardcoded [token type] in [file] with environment variable

### Recommended Next Steps
- Run `/audit-credentials` for a full credential audit including GitHub PATs and OIDC opportunities
- Run `/setup-commit-signing` to sign commits and prevent impersonation
- Run `/setup-oidc` to replace long-lived cloud credentials with OIDC tokens
- Enable GitHub secret scanning push protection on your repository
- Set up a secrets manager (Vault, AWS Secrets Manager, 1Password) for production credentials
```
