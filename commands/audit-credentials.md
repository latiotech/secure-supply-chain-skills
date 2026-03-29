---
description: Find long-lived tokens, hardcoded secrets, and credentials that should be rotated or replaced
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, aws:*, gcloud:*, az:*)
---

Scan the entire project for long-lived credentials, hardcoded secrets, and tokens that should be rotated or replaced with short-lived alternatives. **This command takes action by default** - it flags every finding with severity, location, and the specific fix. Where safe to do so (e.g., adding `.env` to `.gitignore`), it makes changes directly.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/actions-configs.md` for OIDC and PAT hygiene guidance.

## Why this matters

Long-lived credentials are the #1 lateral movement vector in supply chain attacks. The TeamPCP campaign cascaded across npm, PyPI, Docker Hub, and extension marketplaces - all from a single over-privileged, non-expiring PAT. Short-lived, scoped tokens limit the blast radius of any single compromise.

## Scan 1: GitHub Actions - Cloud Credentials (CRITICAL)

Search all `.github/workflows/*.yml` files for long-lived cloud credentials:

### AWS long-lived keys
Look for these patterns - they indicate static AWS keys stored as GitHub Secrets:
- `${{ secrets.AWS_ACCESS_KEY_ID }}`
- `${{ secrets.AWS_SECRET_ACCESS_KEY }}`
- `AWS_ACCESS_KEY_ID` in `env:` blocks
- Any step that sets AWS credentials without using `aws-actions/configure-aws-credentials` with `role-to-assume`

For each finding:
- Flag as **CRITICAL** - these are long-lived, do not auto-expire, and grant whatever IAM permissions are attached
- Show the workflow file, job, and step
- Check if the workflow already has `id-token: write` permission (if so, OIDC may be partially set up)
- Recommend: run `/setup-oidc` to replace with short-lived OIDC tokens

### GCP long-lived keys
Look for:
- `${{ secrets.GCP_SA_KEY }}` or similar service account key references
- `google-github-actions/auth` using `credentials_json` instead of `workload_identity_provider`
- `GOOGLE_APPLICATION_CREDENTIALS` set from a secret
- Base64-encoded service account JSON in secrets

For each finding:
- Flag as **CRITICAL**
- Explain: GCP service account keys don't expire by default and grant full SA permissions
- Recommend: Workload Identity Federation (OIDC) via `/setup-oidc`

### Azure long-lived keys
Look for:
- `${{ secrets.AZURE_CREDENTIALS }}` (the classic JSON blob)
- `${{ secrets.AZURE_CLIENT_SECRET }}`
- `azure/login` without `enable-AzPSSession` and federated credential

For each finding:
- Flag as **CRITICAL**
- Recommend: Federated credentials via `/setup-oidc`

### Other CI/CD credentials
Look for:
- `${{ secrets.NPM_TOKEN }}` - check if npm provenance with OIDC is possible instead
- `${{ secrets.PYPI_TOKEN }}` or `${{ secrets.PYPI_API_TOKEN }}` - check if trusted publisher (OIDC) is possible
- `${{ secrets.DOCKER_PASSWORD }}` or `${{ secrets.DOCKERHUB_TOKEN }}`
- `${{ secrets.SONAR_TOKEN }}`, `${{ secrets.CODECOV_TOKEN }}`, etc.
- Any `${{ secrets.* }}` reference - list them all and categorize as long-lived vs short-lived

For npm/PyPI specifically:
- **npm**: Can use [npm provenance](https://docs.npmjs.com/generating-provenance-statements) with `--provenance` flag - eliminates the need for `NPM_TOKEN` in many cases
- **PyPI**: Can use [Trusted Publishers](https://docs.pypi.org/trusted-publishers/) - OIDC-based, no API token needed

## Scan 2: Hardcoded Secrets in Code and Config (CRITICAL)

Search the entire codebase for hardcoded credentials using pattern matching:

### High-confidence patterns (likely real secrets)
- API key patterns: strings matching `sk-[a-zA-Z0-9]{20,}`, `ghp_[a-zA-Z0-9]{36}`, `gho_[a-zA-Z0-9]{36}`, `github_pat_[a-zA-Z0-9_]{80,}`
- AWS keys: `AKIA[0-9A-Z]{16}`
- Private keys: `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`
- Connection strings: `postgresql://`, `mongodb://`, `mysql://` with embedded passwords
- Generic patterns: `password\s*=\s*['"][^'"]+['"]`, `api_key\s*=\s*['"][^'"]+['"]`, `token\s*=\s*['"][^'"]+['"]` (excluding obvious test/example values)

### Config files to check specifically
- `.env`, `.env.*` files - should never be committed
- `.npmrc` with `_authToken=`
- `.pypirc` with `password =`
- `docker-compose.yml` with hardcoded passwords in environment
- `application.yml` / `application.properties` with credentials
- `terraform.tfvars` with sensitive values

For each finding:
- Show the file, line number, and the pattern matched (redact the actual value in output)
- Classify: is this a test fixture, example, or likely real credential?
- If `.env` files are not in `.gitignore`, add them
- Recommend: use environment variables, secret managers (Vault, AWS Secrets Manager, 1Password), or CI/CD secrets

## Scan 3: Git History for Leaked Secrets (HIGH)

Check if secrets were ever committed to git history:
- Run `git log --diff-filter=A --name-only -- '*.env' '*.pem' '*.key' '*credentials*' '*.tfvars'` to find sensitive files that were added
- If any are found, warn that they're still in git history even if deleted from the working tree
- Recommend: use `git-filter-repo` or BFG Repo Cleaner to remove them, then rotate the exposed credentials

## Scan 4: GitHub PATs and Tokens via API (HIGH)

If the user has `gh` CLI authenticated with admin access:

### List fine-grained PATs
```bash
gh api /orgs/{org}/personal-access-tokens --paginate --jq '.[] | {owner: .owner.login, expires_at, permissions: .permissions, repositories: .repositories}'
```

For each PAT:
- Flag if no expiration is set
- Flag if scoped to all repos (should be specific repos)
- Flag if permissions are broader than needed
- Flag classic tokens (they lack fine-grained scoping)

### Check GitHub App installations
```bash
gh api /orgs/{org}/installations --jq '.[] | {app: .app_slug, permissions, repository_selection}'
```
- Flag apps with `repository_selection: "all"` - should be specific repos
- Flag apps with write permissions they don't need

If `gh` isn't authenticated or doesn't have admin access, skip this scan and note it as a manual step.

## Scan 5: Registry and Package Manager Auth (MEDIUM)

Check for long-lived auth tokens in package manager configs:

- `.npmrc`: Look for `_authToken=` or `_auth=` - these are typically long-lived
- `.yarnrc.yml`: Look for `npmAuthToken`
- `~/.pypirc` or project `.pypirc`: Look for `password =`
- `~/.docker/config.json`: Check for `credsStore` (good) vs plain `auth` (bad - base64-encoded credentials)
- `~/.kube/config`: Check for embedded credentials vs exec-based auth

For each:
- Recommend credential helpers or short-lived alternatives
- Check if the file is in `.gitignore`

## Output

Generate a credential audit report:

```
## Credential Audit Report

**Project**: [name]
**Date**: [date]
**Total findings**: N

### Summary

| Category | Critical | High | Medium | OIDC Available? |
|----------|----------|------|--------|-----------------|
| CI/CD Cloud Credentials | 2 | 0 | 0 | Yes - /setup-oidc |
| Hardcoded Secrets | 0 | 1 | 0 | N/A |
| Git History Leaks | 0 | 1 | 0 | N/A |
| GitHub PATs | 0 | 0 | 2 | N/A |
| Registry Auth | 0 | 0 | 1 | Partial |

### Critical: Long-Lived Cloud Credentials in CI/CD

| Workflow | Secret | Used For | Replace With |
|----------|--------|----------|-------------|
| deploy.yml:23 | AWS_ACCESS_KEY_ID | S3 deploy | OIDC (`/setup-oidc`) |
| publish.yml:45 | NPM_TOKEN | npm publish | Provenance (`npm publish --provenance`) |

### High: Secrets in Git History
[Findings with remediation steps]

### Medium: Long-Lived Registry Auth
[Findings with remediation steps]

### Changes Made
- [x] Added .env* to .gitignore
- [x] ...

### Recommended Next Steps
1. Run `/setup-oidc` to replace [N] long-lived cloud credentials with OIDC tokens
2. Rotate [credential] - it was found in git history
3. Switch npm publishing to use `--provenance` (eliminates NPM_TOKEN)
4. Switch PyPI publishing to use Trusted Publishers (eliminates PYPI_TOKEN)
5. Set expiration on [N] GitHub PATs
```

Save the report to `credential-audit.md` in the project root.
