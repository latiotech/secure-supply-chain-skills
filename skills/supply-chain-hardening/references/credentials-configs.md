# Credential Hardening Configurations

Practical configs for scanning, preventing, and managing secrets across your project.

## Secret Scanning with Betterleaks

```bash
# Install betterleaks
pip install betterleaks

# Scan current directory
betterleaks --path .

# Scan git history for previously committed secrets
betterleaks --path . --scan-history

# Scan with custom config
betterleaks --path . --config .betterleaks.toml
```

### .betterleaks.toml - Custom rules

```toml
# Allowlist paths that contain test fixtures or examples
[allowlist]
paths = [
    "test/fixtures/",
    "**/*_test.go",
    "**/*.test.ts",
    "docs/examples/",
]

# Allowlist specific patterns (e.g., placeholder values)
regexes = [
    "EXAMPLE_KEY",
    "your-api-key-here",
    "changeme",
    "password123",
]
```

## Pre-commit Hook Setup

### Using pre-commit framework with gitleaks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

```bash
# Install and activate
pip install pre-commit
pre-commit install

# Run against all files (first-time scan)
pre-commit run gitleaks --all-files
```

### Standalone git hook (no framework)

```bash
#!/bin/sh
# .git/hooks/pre-commit
# Requires betterleaks or gitleaks to be installed

if command -v betterleaks &> /dev/null; then
    betterleaks --path . --staged-only
    if [ $? -ne 0 ]; then
        echo "Secret detected in staged files. Commit blocked."
        exit 1
    fi
elif command -v gitleaks &> /dev/null; then
    gitleaks protect --staged
    if [ $? -ne 0 ]; then
        echo "Secret detected in staged files. Commit blocked."
        exit 1
    fi
fi
```

## .gitignore Patterns for Secrets

```gitignore
# Secrets and credentials - never commit these
.env
.env.*
!.env.example
!.env.test

# Private keys and certificates
*.pem
*.key
*.p12
*.pfx
*.jks
*.keystore

# Package manager auth
.npmrc
.pypirc
.docker/config.json

# Cloud credentials
.aws/credentials
.azure/
.config/gcloud/

# Terraform secrets
terraform.tfvars
*.auto.tfvars
.terraform/

# IDE secrets (sometimes contain tokens)
.vscode/settings.json
.idea/workspace.xml
```

## .env.example Template

```bash
# .env.example - Copy to .env and fill in real values
# NEVER commit .env with real credentials

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# API Keys
API_KEY=your-api-key-here
SECRET_KEY=generate-a-random-secret

# Cloud (prefer OIDC in CI/CD - see /setup-oidc)
AWS_REGION=us-east-1
# AWS_ACCESS_KEY_ID=        # Use OIDC instead
# AWS_SECRET_ACCESS_KEY=    # Use OIDC instead

# Third-party services
STRIPE_SECRET_KEY=sk_test_...
SENDGRID_API_KEY=SG.your-key-here
```

## GitHub Secret Scanning Push Protection

Enable push protection to block commits containing secrets before they reach the remote:

1. Go to **Settings → Code security → Secret scanning**
2. Enable **Secret scanning**
3. Enable **Push protection**
4. Optionally enable **Validity checks** (GitHub checks if detected secrets are still active)

### For organizations

```bash
# Enable secret scanning for all repos in the org
gh api -X PUT /orgs/YOURORG/code-security/configurations \
  --field secret_scanning=enabled \
  --field secret_scanning_push_protection=enabled
```

## PAT and Service Account Hygiene

Best practices for PAT and service account management:

1. **Use fine-grained PATs** instead of classic tokens
2. **Scope to specific repos** - never org-wide
3. **Set expiration** - 90 days maximum
4. **Use minimum permissions** - read-only unless write is explicitly needed
5. **Audit regularly** - `gh api /orgs/YOURORG/personal-access-tokens` to list all PATs
6. **Prefer GitHub Apps** over PATs for automation - apps have granular permissions and are auditable
7. **Use GITHUB_TOKEN** where possible - scoped to the current repo and expires after the job

### Audit existing PATs

```bash
# List all fine-grained PATs in org (requires admin)
gh api /orgs/YOURORG/personal-access-tokens --paginate

# List classic PATs (admin settings only - no API)
# Go to: https://github.com/organizations/YOURORG/settings/personal-access-tokens/active
```

## Removing Secrets from Git History

If a secret was committed to git history, rotating the credential is step one. Then clean the history:

### Using git-filter-repo (recommended)

```bash
# Install
pip install git-filter-repo

# Remove a specific file from all history
git filter-repo --invert-paths --path .env --force

# Remove a specific string from all history
git filter-repo --replace-text <(echo 'AKIA1234567890ABCDEF==>REDACTED') --force
```

### Using BFG Repo Cleaner

```bash
# Remove all files matching a pattern
java -jar bfg.jar --delete-files '*.env' repo.git

# Replace specific strings
echo 'AKIA1234567890ABCDEF' > passwords.txt
java -jar bfg.jar --replace-text passwords.txt repo.git
```

### After history rewriting

1. Force push the cleaned history: `git push --force --all`
2. Force push cleaned tags: `git push --force --tags`
3. All collaborators must re-clone (not just pull)
4. Contact GitHub support to clear cached views
5. **Most importantly**: rotate the exposed credential - it's already compromised

## Credential Rotation Checklist

When a secret is leaked, assume lateral movement. Rotate in this order:

1. **The leaked credential itself** - revoke and regenerate immediately
2. **Anything the credential could access** - if an npm token leaked, check for published packages
3. **Adjacent credentials** - if a CI token leaked, rotate all secrets in that CI environment
4. **Downstream services** - API keys, database passwords, cloud credentials accessible from the compromised context

### Cloud-specific rotation

```bash
# AWS - rotate access keys
aws iam create-access-key --user-name USERNAME
aws iam delete-access-key --user-name USERNAME --access-key-id OLD_KEY_ID

# GitHub - regenerate PAT (no CLI - must use web UI)
# https://github.com/settings/tokens

# npm - regenerate token
npm token revoke TOKEN_ID
npm token create
```
