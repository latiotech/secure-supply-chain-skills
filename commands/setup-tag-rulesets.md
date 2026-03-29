---
description: "Walkthrough: Protect release tags from force-push attacks with GitHub rulesets"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*)
---

Interactive walkthrough to set up GitHub tag rulesets that prevent force-push and deletion attacks on release tags. This is the mitigation for the TeamPCP attack vector where 110+ tags were hijacked via force-push.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/actions-configs.md` for context on the TeamPCP attack.

## Why tag rulesets?

In March 2026, the TeamPCP campaign compromised GitHub Actions across 110+ repos by force-pushing malicious code to existing version tags (e.g., `v1`, `v4.2.0`). Anyone using `actions/checkout@v4` got the attacker's code. Tag rulesets block force-pushes and deletions, making this attack impossible.

## Step 1: Assess current state

Check the project for:
- Existing tag patterns (run `git tag --list` to see what tags exist)
- Whether GitHub branch protection or rulesets are already configured (check with `gh api repos/{owner}/{repo}/rulesets`)
- Whether Dependabot or Renovate is configured for GitHub Actions updates
- The project's release workflow and tag naming convention

Present findings and ask:
- What tag pattern do you use for releases? (e.g., `v*`, `release-*`, semver only)
- Do you have a release automation account or bot that needs to create/update tags?

## Step 2: Create the tag ruleset

Walk the user through creating the ruleset. This must be done in the GitHub UI or via API - it cannot be configured from code alone.

### Option A: Via GitHub UI

Provide step-by-step instructions:
1. Go to **Settings → Rules → Rulesets → New ruleset → New tag ruleset**
2. Name: `protect-release-tags`
3. Enforcement: **Active**
4. Target tags → Add pattern: `v*` (or their tag pattern)
5. Enable these rules:
   - **Block force pushes** - prevents overwriting existing tags with different commits
   - **Block deletions** - prevents delete-and-recreate attacks
6. Bypass list: Add only the release automation account (if any)
7. Click **Create**

### Option B: Via GitHub API

If they prefer the API, provide the command:
```bash
gh api repos/{owner}/{repo}/rulesets \
  --method POST \
  --field name="protect-release-tags" \
  --field enforcement="active" \
  --field target="tag" \
  --field 'conditions[ref_name][include][]=refs/tags/v*' \
  --field 'rules[][type]=deletion' \
  --field 'rules[][type]=non_fast_forward'
```

Ask for their repo owner/name and tag pattern to generate the exact command.

## Step 3: Verify the ruleset

After creation, verify it works:
1. Try to force-push a test tag: `git tag -f test-ruleset-tag HEAD~1 && git push -f origin test-ruleset-tag`
2. Confirm it's rejected with a protection rules error
3. Clean up the test tag

## Step 4: Organization-level rulesets (if applicable)

If the user manages multiple repos, explain:
- On GitHub Enterprise Cloud, rulesets can be applied at the **organization level** so all repos inherit the protection
- Go to **Organization Settings → Rules → Rulesets → New ruleset**
- This is the recommended approach for orgs with many repos using GitHub Actions

## Step 5: Complementary protections

After tag rulesets are in place, suggest:
- Pin all Actions to commit SHAs (run `/harden-actions` if not done)
- Set up Dependabot to keep pinned SHAs current
- Consider enabling GitHub's Actions allow-list to restrict which Actions can run in the org
