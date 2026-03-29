# GitHub Actions Hardening Configurations

Practical configs for securing GitHub Actions workflows.

## Pin Actions to Commit SHAs

```yaml
# BAD - tags can be force-pushed
- uses: actions/checkout@v4

# GOOD - pinned to immutable SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

### Resolve the correct SHA

Most release tags are **annotated tags**, which have two SHAs — the tag object and the commit. Actions `uses:` needs the **commit SHA**. Use the dereferenced ref:

```bash
# Dereference annotated tag to get the commit SHA (not the tag object SHA)
git ls-remote https://github.com/actions/checkout.git "refs/tags/v4.1.1^{}"
# → b4ffde65f46336ab88eb53be808477a3936bae11

# If empty result (lightweight tag), the non-dereferenced ref is already the commit SHA:
git ls-remote https://github.com/actions/checkout.git refs/tags/v4.1.1
```

WARNING: Using `refs/tags/{tag}` without `^{}` on annotated tags returns the **tag object SHA**, not the commit SHA. Pinning to the tag object SHA will fail at runtime.

### Automate SHA pinning

```bash
# Use StepSecurity's tool to pin all Actions in a workflow (handles dereferencing)
npx pin-github-action .github/workflows/*.yml

# Or use the web UI
# https://app.stepsecurity.io/secureworkflow
```

### Use Renovate or Dependabot to keep pinned SHAs current

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Set Explicit Permissions

```yaml
# Set restrictive default at the workflow level
permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    # Override per-job only where needed
    permissions:
      contents: read
      packages: write  # only if publishing packages
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

### Permission reference - least privilege per task

| Task | Permissions needed |
|------|-------------------|
| Checkout code | `contents: read` |
| Push code | `contents: write` |
| Create PR | `contents: write`, `pull-requests: write` |
| Publish package | `packages: write` |
| Deploy via OIDC | `id-token: write` |
| Comment on issues | `issues: write` |
| Read org members | `members: read` |

## Audit for Dangerous Triggers

### The `pull_request_target` problem

`pull_request_target` runs workflow code from the *base branch* but with *repository secrets*, and the PR can modify the checkout:

```yaml
# DANGEROUS - if this workflow checks out PR code, attackers get your secrets
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # THIS IS THE DANGEROUS PATTERN - checking out PR head with secrets available
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      # Attacker's code now runs with access to secrets
```

### Safe alternative patterns

```yaml
# SAFE - use pull_request (no secrets, runs in fork context)
on:
  pull_request:
    types: [opened, synchronize]

# SAFE - if you MUST use pull_request_target, never checkout PR code
on:
  pull_request_target:
    types: [labeled]
jobs:
  build:
    if: contains(github.event.pull_request.labels.*.name, 'safe-to-test')
    runs-on: ubuntu-latest
    steps:
      # Only checkout base branch code, NOT the PR
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      # Process PR metadata only, never execute PR code
```

### Scan with Zizmor

```bash
# Install
pip install zizmor

# Scan all workflows
zizmor .github/workflows/

# Zizmor detects:
# - pull_request_target with checkout of PR code
# - Unpinned actions (tag references)
# - Overly broad permissions
# - Script injection via untrusted inputs
# - Use of deprecated commands
```

## Enable Tag Rulesets

Prevent force-pushes to release tags:

1. Go to **Settings → Rules → Rulesets → New ruleset**
2. Select **Tag** as the target
3. Set target pattern: `v*` (or your release tag pattern)
4. Enable rules:
   - **Block force pushes** - prevents overwriting existing tags
   - **Block deletions** - prevents delete-and-recreate attacks
5. Set bypass list to only your release automation account
6. On Enterprise Cloud: apply at org level so all repos inherit

## OIDC for Cloud Deployments

Replace long-lived secrets with short-lived OIDC tokens:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-deploy
          aws-region: us-east-1
          # No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY needed
```

### OIDC setup for major clouds

| Cloud | Action | Docs |
|-------|--------|------|
| AWS | `aws-actions/configure-aws-credentials` | [GitHub OIDC for AWS](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) |
| GCP | `google-github-actions/auth` | [GitHub OIDC for GCP](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform) |
| Azure | `azure/login` | [GitHub OIDC for Azure](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-azure) |

## CODEOWNERS for Workflow Protection

```
# .github/CODEOWNERS
# Require security team review for workflow changes
.github/workflows/ @yourorg/security-team
.github/actions/ @yourorg/security-team
.github/dependabot.yml @yourorg/security-team
```

## PAT and Service Account Hygiene

Best practices for PAT and service account hygiene:

1. **Use fine-grained PATs** instead of classic tokens
2. **Scope to specific repos** - never org-wide
3. **Set expiration** - 90 days maximum
4. **Use the minimum permissions** - read-only unless write is explicitly needed
5. **Audit regularly** - `gh api /orgs/YOURORG/personal-access-tokens` to list all PATs
6. **Prefer GitHub Apps** over PATs for automation - apps have granular permissions and are auditable
7. **Use GITHUB_TOKEN** where possible - it's scoped to the current repo and expires after the job

### Audit existing PATs

```bash
# List all fine-grained PATs in org (requires admin)
gh api /orgs/YOURORG/personal-access-tokens --paginate

# List classic PATs (admin settings only - no API for this)
# Go to: https://github.com/organizations/YOURORG/settings/personal-access-tokens/active
```

## Runtime Detection on CI Runners

### StepSecurity Harden Runner

```yaml
steps:
  - name: Harden Runner
    uses: step-security/harden-runner@17d0e2bd7d51742c71671bd19fa12bdc9d40a3d6 # v2.8.1
    with:
      egress-policy: audit  # Start with audit, move to block
      # block mode: only allow specific endpoints
      # allowed-endpoints: >
      #   github.com:443
      #   registry.npmjs.org:443
```

### Falco rules for CI runner monitoring

```yaml
# Detect /proc/mem dumps
- rule: Dump Memory using /proc/ Filesystem
  desc: Detect reading from /proc/[pid]/mem
  condition: >
    open_read and fd.name startswith /proc/ and fd.name contains /mem
  output: >
    Memory dump detected (user=%user.name command=%proc.cmdline file=%fd.name)
  priority: CRITICAL

# Detect credential file sweeps
- rule: Read Sensitive Credential Files
  desc: Detect reading SSH keys, cloud creds, .env files
  condition: >
    open_read and (fd.name startswith /home/ or fd.name startswith /root/)
    and (fd.name contains .ssh or fd.name contains .aws or fd.name contains .env)
  output: >
    Sensitive file read (user=%user.name file=%fd.name command=%proc.cmdline)
  priority: WARNING
```
