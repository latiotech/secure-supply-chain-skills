# Supply Chain Security Plugin

A Claude Code plugin for auditing and hardening software supply chains.

## Structure

```
.claude-plugin/plugin.json   - Plugin metadata
commands/                     - Slash commands (one .md per command)
  audit-supply-chain.md       - Full audit + auto-fix (entry point)
  audit-credentials.md        - Credential and token audit
  update-pins.md              - Update pinned deps/actions/images to latest versions
  minimize.md                 - Remove unused deps and convert to multi-stage Docker builds
  harden-*.md                 - Domain-specific action commands
  setup-*.md                  - Interactive walkthrough commands
skills/supply-chain-hardening/
  SKILL.md                    - Skill definition and trigger config
  references/                 - Checklist, tool catalog, and implementation configs
```

## Two Types of Commands

**Action commands** (`audit-*`, `harden-*`) take action by default - they scan, fix, and explain. They use the "Actions to Take" structure: Detection → ordered fixes by severity → summary of changes made + manual steps remaining.

**Walkthrough commands** (`setup-*`) are interactive guides for configurations that require steps outside the codebase (cloud provider setup, GitHub settings, Kubernetes config). They explain each step, provide the commands to run, and tell the user what to verify.

## Command Frontmatter

Commands have YAML frontmatter with:
- `description` - Short description shown in command list
- `allowed-tools` - Tools the command can use (Read, Write, Edit, Glob, Grep, plus specific Bash patterns)
- `argument-hint` (optional) - Hint for the argument the command accepts

Commands reference skill materials via `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/`.

## Conventions

- Action commands **make changes by default** and explain each one - they don't ask before every change
- Dangerous changes (removing triggers, deleting provisioners) are flagged but NOT auto-applied
- All commands group findings by severity: CRITICAL → HIGH → MEDIUM
- `allowed-tools` should use Claude Code's built-in tools (Read, Glob, Grep) instead of shell equivalents (cat, ls, find, head)
- Each action command ends with a structured summary: Changes Made / Manual Steps Needed / Recommended Next Steps
- Action commands point to relevant walkthrough commands in their "Recommended Next Steps"
- Reference material links in `tools.md` should point to `checklist.md` (not the old `supply-chain-security-checklist.md` name)

## Adding a New Domain

1. Create `commands/harden-<domain>.md` following the action command structure
2. Add checks to `commands/audit-supply-chain.md` under Phase 2
3. Add the command to the report template in Phase 4 of `audit-supply-chain.md`
4. If the domain needs implementation configs, add `references/<domain>-configs.md`
5. Update `README.md` - action command table, "What Gets Checked" details section
6. Add checklist items to `references/checklist.md`
7. Add tools to `references/tools.md`
8. If the domain has advanced setup steps, create `commands/setup-<topic>.md` walkthrough commands
9. Update `SKILL.md` trigger description if new keywords are needed
