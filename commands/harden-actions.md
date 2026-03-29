---
description: Pin GitHub Actions to SHAs, fix permissions, and flag dangerous triggers
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, zizmor:*, pip:*, npx:*, curl:*, which:*, brew:*, cargo:*)
---

Audit and harden GitHub Actions workflows for supply chain security. **This command takes action by default** - it pins Actions to commit SHAs, sets explicit permissions, and fixes script injection. Changes are explained as they are made.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/actions-configs.md` for detailed configurations and context.

## Detection

Find all workflow files:
- `.github/workflows/*.yml`
- `.github/workflows/*.yaml`
- `.github/actions/*/action.yml` (composite actions)

If no workflow files are found, report that and exit.

## Prerequisites

Workflow files were found. Check if `zizmor` is installed (`which zizmor`).

If **not installed**, tell the user:

> **Recommended: install zizmor** — a purpose-built GitHub Actions security scanner that catches misconfigurations, injection vulnerabilities, and privilege issues that pattern-matching alone misses.
>
> ```bash
> # macOS
> brew install zizmor
> # or with cargo
> cargo install zizmor
> # or with pip
> pip install zizmor
> ```
>
> Once installed, re-run this command for scanner-driven results. Continuing with pattern-based analysis for now.

## Actions to Take

Work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Run Zizmor (CRITICAL — if installed)

If zizmor is installed, run it against all workflow files:

```bash
zizmor .github/workflows/
```

Parse and report findings grouped by severity. Use Zizmor's output to prioritize the fixes below — address scanner findings first before moving to pattern-based checks. If Zizmor already flagged an issue that a later step would catch, note it as "confirmed by Zizmor" and fix it in this step.

### 2. Pin all Actions to commit SHAs (CRITICAL)

Scan all `uses:` directives for Actions referenced by tag (`@v4`, `@main`) instead of full commit SHA.

For each unpinned action:

**Important: resolve the commit SHA, not the tag object SHA.** Most releases use annotated tags, which have two SHAs — the tag object and the commit it points to. GitHub Actions `uses:` requires the **commit SHA**. Use the dereferenced (`^{}`) ref:

1. **Preferred** — dereference the tag to get the commit SHA:
   ```bash
   git ls-remote https://github.com/{owner}/{repo}.git "refs/tags/{tag}^{}"
   ```
   If this returns empty (lightweight tag), fall back to `refs/tags/{tag}` which is already the commit SHA.

2. **Alternative** — use the GitHub API:
   ```bash
   gh api repos/{owner}/{repo}/git/ref/tags/{tag} --jq '.object.sha'
   ```

3. If both fail (network issue, private repo, typo), add a `# TODO: pin to SHA` comment. **Never fabricate a SHA.**

- Replace the tag reference with the full commit SHA, keeping the tag as a comment: `actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1`
- Explain: tags can be force-pushed to point at malicious code (this is exactly how the TeamPCP attack worked on 110+ repos)

### 3. Set explicit permissions (HIGH)

For each workflow file:
- If there is no top-level `permissions:` key, add `permissions: contents: read` as a restrictive default
- If `permissions: write-all` is set, replace with the minimum permissions needed based on what the steps actually do
- Use the permission reference table from the actions-configs reference

### 4. Flag and fix dangerous triggers (CRITICAL)

Search for `pull_request_target` triggers. For each:
- Check if the workflow checks out PR head code (`ref: ${{ github.event.pull_request.head.sha }}`)
- If it does, this is the exact TeamPCP attack pattern - **flag it prominently** and suggest switching to `pull_request` trigger or removing the PR head checkout
- Do NOT auto-fix these - explain the risk and let the user decide, as changing triggers affects workflow behavior

### 5. Fix script injection (HIGH)

Search for `${{ github.event.* }}` used directly in `run:` blocks. These are injectable:
- `${{ github.event.pull_request.title }}`
- `${{ github.event.issue.body }}`
- `${{ github.event.comment.body }}`

For each finding, replace with an environment variable pattern:
```yaml
env:
  PR_TITLE: ${{ github.event.pull_request.title }}
run: echo "$PR_TITLE"
```

### 6. Add CODEOWNERS for workflow protection (MEDIUM)

If `.github/CODEOWNERS` doesn't exist or doesn't cover `.github/workflows/`:
- Create or update CODEOWNERS to add a rule for `.github/workflows/` and `.github/actions/`
- Add a commented placeholder for the team name: `# .github/workflows/ @yourorg/security-team`

### 7. Set up Dependabot for Actions updates

If `.github/dependabot.yml` doesn't exist, create it with a github-actions ecosystem entry to keep pinned SHAs current:
```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

If it exists but doesn't include github-actions, add the entry.

## Output

After making all changes, provide a summary:

```
## GitHub Actions Hardening Summary

### Changes Made
- [x] Pinned N actions to commit SHAs across M workflow files
- [x] Added explicit permissions to N workflows
- [x] Fixed N script injection vulnerabilities
- [x] Added/updated CODEOWNERS
- [x] Added Dependabot config for Actions updates

### Requires Manual Attention
- [ ] pull_request_target in [workflow] - review and decide on fix
- [ ] Long-lived AWS credentials in [workflow] - run `/setup-oidc` to replace with OIDC
- [ ] Enable tag rulesets - run `/setup-tag-rulesets` for walkthrough

### Recommended Next Steps
- Run `/setup-oidc` to replace long-lived cloud credentials with OIDC tokens
- Run `/setup-tag-rulesets` to protect release tags from force-push attacks
- Run `/setup-runner-monitoring` to add runtime detection to CI runners
```
