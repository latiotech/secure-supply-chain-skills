---
description: Audit IDE extensions and secure developer tool configs
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(code:*, osquery*:*, git:*)
---

Audit IDE extensions and developer tool configurations for supply chain security. **This command takes action by default** - it cleans up extension configs, adds devcontainer isolation, and flags risks. Changes are explained as they are made.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/checklist.md` for the full checklist (IDE Extensions section).
Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/tools.md` for tool options.

## Detection

Find IDE and developer tool configuration files in the project:

- `.vscode/extensions.json` → Recommended VS Code extensions
- `.vscode/settings.json` → VS Code workspace settings
- `.devcontainer/devcontainer.json` / `devcontainer.json` → Dev container configs
- `.cursor/` → Cursor editor configs

## Actions to Take

Work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Audit recommended extensions (HIGH)

If `.vscode/extensions.json` exists, review the `recommendations` array:
- Flag any extensions that are not from verified publishers
- Flag extensions with very low install counts (note this requires manual verification)
- Remove any extensions from the recommendations that are clearly unnecessary for the project
- Add a comment block at the top of the file documenting the last audit date

### 2. Check for secrets in workspace settings (HIGH)

Scan `.vscode/settings.json` and other IDE config files for:
- Hardcoded API keys, tokens, or credentials
- References to local credential files
- Sensitive environment variable values

For each finding:
- Remove the secret value
- Replace with a placeholder or reference to a secrets manager
- Add the file to `.gitignore` if it contains user-specific settings

### 3. Add or harden devcontainer config (MEDIUM)

If `.devcontainer/devcontainer.json` doesn't exist:
- Create a basic devcontainer config that:
  - Uses a project-appropriate base image
  - Runs as a non-root user (`"remoteUser": "vscode"`)
  - Lists only the verified extensions needed for the project
  - Explain: devcontainers isolate extension execution from the host machine

If it exists:
- Verify it runs as non-root
- Verify extensions listed are verified and necessary

### 4. Create extension allowlist documentation (MEDIUM)

If no extension policy exists in the repo, add a commented section to `.vscode/extensions.json` or create a note explaining:
- Which extensions are approved for this project
- How to vet new extensions before installing
- That extensions should be reviewed for excessive permissions (network, filesystem access)

## Output

After making all changes, provide a summary:

```
## IDE Extension Hardening Summary

### Changes Made
- [x] Audited N recommended extensions in extensions.json
- [x] Removed secrets from .vscode/settings.json
- [x] Created/hardened devcontainer config
- [x] Added extension policy documentation

### Manual Steps Needed
- [ ] Verify extension [name] is from a trusted publisher (low install count)
- [ ] Set up VS Code Profiles for team standardization
- [ ] Audit installed extensions across team with osquery

### Recommended Next Steps
- Use VS Code Profiles to enforce consistent extension sets
- Consider remote dev environments (Codespaces/Gitpod) for stronger isolation
```
