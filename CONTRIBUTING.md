# Contributing

Thanks for your interest in improving this plugin. Whether it's a bug report, a new check, or a scanner integration, contributions are welcome.

## Reporting Issues

Open an issue at [github.com/latiotech/secure-supply-chain-skills/issues](https://github.com/latiotech/secure-supply-chain-skills/issues). Helpful things to include:

- Which command you ran (`/audit-supply-chain`, `/harden-actions`, etc.)
- What happened vs what you expected
- Your project's relevant stack (e.g., "npm + GitHub Actions + Docker")
- Whether a scanner was installed and which version

False positives, missed findings, and broken workflows are all valid issues.

## Project Structure

```
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

## Adding or Improving a Check

1. Identify which command the check belongs to (or if a new command is needed).
2. Follow the existing command structure — see `CLAUDE.md` for conventions.
3. If the check relies on a scanner, it should work without the scanner too (pattern-based fallback).
4. Test against a real project if possible.

## Adding a New Domain

Follow the steps in `CLAUDE.md` under "Adding a New Domain". In short:

1. Create `commands/harden-<domain>.md`
2. Add checks to `audit-supply-chain.md`
3. Add reference material to `references/`
4. Update `SKILL.md`, `README.md`, and `CLAUDE.md`

## Adding or Updating a Scanner Integration

Each domain has one recommended scanner. If you're proposing a replacement or addition:

- Explain why the proposed tool is better for this plugin's use case (accuracy, speed, false-positive rate, install simplicity)
- The scanner should be free and open source
- The command should detect the relevant file types first, then suggest the scanner install only if those files exist
- The scanner runs as step 1 in the command flow, with pattern-based checks as fallback

Current scanner mapping:

| Domain | Scanner |
|--------|---------|
| GitHub Actions | zizmor |
| Credentials | betterleaks |
| Containers | hadolint |
| IaC | checkov |
| AI/ML | modelscan |

## Command Conventions

- **Action commands** (`audit-*`, `harden-*`, `update-*`) make changes by default and explain each one
- **Walkthrough commands** (`setup-*`) are interactive guides for external setup
- All commands group findings by severity: CRITICAL > HIGH > MEDIUM
- Each action command ends with a structured summary: Changes Made / Manual Steps / Next Steps
- Pins must be resolved deterministically — use `docker buildx imagetools inspect` (not `docker pull`), dereference annotated tags with `^{}`, query registry APIs
- Never fabricate a version, SHA, or digest

## Pull Request Guidelines

- Keep PRs focused — one domain or one check per PR is easier to review
- Include a brief description of what the change does and why
- If you're adding a new check, describe what attack or misconfiguration it catches
- Test against at least one real repo before submitting

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
