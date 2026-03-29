---
description: Scan for leaked secrets, set up pre-commit hooks, and harden credential hygiene
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, betterleaks:*, gitleaks:*, trufflehog:*, pip:*, npx:*, curl:*)
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

## Actions to Take

Work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Run Betterleaks to scan for secrets (CRITICAL)

Check if betterleaks is installed (`which betterleaks`). If available:

```bash
betterleaks --path .
```

Report all findings grouped by severity. For each leaked secret:
- Show the file, line number, and rule that matched (redact the actual secret value)
- Classify: test fixture, example/placeholder, or likely real credential
- If it's a likely real credential, flag as **CRITICAL** and recommend immediate rotation

If betterleaks is not installed, fall back to pattern-based scanning (see step 2). Mention betterleaks as the recommended tool for comprehensive scanning.

### 2. Pattern-based secret scanning (CRITICAL)

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

### 5. Set up pre-commit hook for secret detection (HIGH)

If a `.pre-commit-config.yaml` exists, check if it includes a secret scanning hook. If not, add one:

```yaml
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

If `.pre-commit-config.yaml` doesn't exist but `.git/hooks/` does, create a basic pre-commit hook or recommend setting up pre-commit framework. **Do NOT overwrite an existing pre-commit hook.**

If neither exists, recommend installing pre-commit:
```bash
pip install pre-commit
pre-commit install
```

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
- [x] Added gitleaks pre-commit hook to .pre-commit-config.yaml
- [x] Flagged N auth tokens in package manager configs

### Requires Manual Attention
- [ ] Rotate [credential] found in [file] - it may be compromised
- [ ] Remove secrets from git history using git-filter-repo
- [ ] Rotate [N] credentials found in git history
- [ ] Replace hardcoded [token type] in [file] with environment variable

### Recommended Next Steps
- Run `/audit-credentials` for a full credential audit including GitHub PATs and OIDC opportunities
- Run `/setup-oidc` to replace long-lived cloud credentials with OIDC tokens
- Enable GitHub secret scanning push protection on your repository
- Set up a secrets manager (Vault, AWS Secrets Manager, 1Password) for production credentials
```
