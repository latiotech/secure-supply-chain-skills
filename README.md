# Supply Chain Security Plugin for Claude Code

Protect your projects from supply chain attacks with Claude.

## What Is This?

When you build software, you rely on hundreds of third-party packages, container images, GitHub Actions, and more. Attackers increasingly target these dependencies to sneak malicious code into otherwise trustworthy projects. This is called a **supply chain attack**.

This plugin gives Claude Code the ability to **audit your project** for supply chain risks and **fix them for you**, covering seven areas:

| Area | Examples of What Could Go Wrong |
|------|-------------------------------|
| **Packages** (npm, pip, etc.) | A dependency gets hijacked and runs a malicious install script |
| **Containers** (Docker) | Your base image silently changes to include a backdoor |
| **GitHub Actions** | A third-party Action gets compromised and steals your secrets |
| **Infrastructure as Code** (Terraform) | An unverified module provisions resources you didn't ask for |
| **AI/ML Models** | A pickle file executes arbitrary code when loaded |
| **IDE Extensions** | A VS Code extension exfiltrates code from your workspace |
| **Credentials** | A leaked API key or long-lived token enables lateral movement across services |

Based on the [Latio](https://latio.com) Supply Chain Security Checklist

## Getting Started

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and working

### Installation

Install the plugin by running this in your terminal:

```bash
claude install-plugin https://github.com/confusedcrib/secure-supply-chain-skills
```

### Your First Audit

Once installed, open Claude Code in any project and run:

```
/audit-supply-chain
```

This scans your repo, figures out which areas apply to you, **auto-fixes critical items** (pinning versions, resolving SHAs, disabling install scripts), and gives you a report showing what was fixed and what still needs attention.

## Two Types of Commands

### Action Commands - Fix Issues Directly

These commands **take action by default**. They scan your repo, make changes (pin versions to exact hashes, resolve commit SHAs, fix configs), and explain each change as it's made. Run these first.

| Command | What It Does |
|---------|-------------|
| `/audit-supply-chain` | Full audit - detects what's in your repo, checks everything, auto-fixes critical items |
| `/harden-packages` | Pin dependency versions, disable install scripts, secure registry configs |
| `/harden-containers` | Pin base images by digest, enforce non-root, create .dockerignore |
| `/harden-actions` | Pin Actions to SHAs, set permissions, fix script injection, add Dependabot |
| `/harden-iac` | Pin Terraform modules/providers, generate lockfiles, flag provisioners |
| `/harden-ai-ml` | Fix unsafe torch.load/pickle.load, pin model sources, add hash verification |
| `/harden-ide-extensions` | Audit extensions, remove secrets from settings, add devcontainer config |
| `/harden-credentials` | Scan for leaked secrets, set up pre-commit hooks, harden .gitignore, fix credential anti-patterns |
| `/audit-credentials` | Find long-lived tokens, hardcoded secrets, credentials that should be rotated or replaced with OIDC |
| `/update-pins` | Check pinned deps, Actions, images, and modules for newer versions — auto-updates patch/minor, flags majors with changelogs |
| `/minimize` | Remove unused dependencies and convert Dockerfiles to multi-stage builds to reduce attack surface |

### Walkthrough Commands - Guided Setup for Advanced Items

These commands are **interactive walkthroughs** for configurations that require steps outside your codebase (cloud provider setup, Kubernetes config, GitHub settings). Run these separately when you're ready to tackle advanced hardening.

| Command | What It Walks You Through |
|---------|--------------------------|
| `/setup-oidc` | Replace long-lived cloud credentials with OIDC tokens (AWS, GCP, Azure) |
| `/setup-image-signing` | Sign container images with Cosign/Sigstore |
| `/setup-tag-rulesets` | Protect release tags from force-push attacks (GitHub rulesets) |
| `/setup-admission-control` | Enforce image policies in Kubernetes (Kyverno/OPA) |
| `/setup-sbom` | Generate SBOMs and SLSA provenance attestations |
| `/setup-commit-signing` | Set up commit and tag signing (SSH, GPG, or Sigstore gitsign) |
| `/setup-runner-monitoring` | Add runtime detection to CI runners (StepSecurity/Falco) |

## How It Works

1. **Detection** - Commands look at your repo's files to figure out which domains apply (e.g., if you have a `package.json`, it checks packages; if you have a `Dockerfile`, it checks containers).
2. **Analysis** - For each relevant domain, checks are run ordered by severity.
3. **Action** - Critical issues are fixed directly: versions pinned, SHAs resolved, configs secured. Each change is explained.
4. **Report** - A summary shows what was fixed, what needs manual attention, and which walkthrough commands to run next.

## What Gets Checked and Fixed

<details>
<summary><strong>Packages</strong> - npm, pnpm, yarn, pip, uv, Go modules, Cargo, Bundler, Maven, Gradle</summary>

- Are dependency versions pinned (no floating ranges)?
- Are install scripts disabled where possible?
- Is the lockfile present and up to date?
- Are registry scopes configured to prevent dependency confusion?
</details>

<details>
<summary><strong>Containers</strong> - Docker, OCI images</summary>

- Are base images pinned by digest (not just a mutable tag)?
- Do containers run as non-root?
- Are multi-stage builds used to minimize the final image?
- Are images signed and verified?
</details>

<details>
<summary><strong>GitHub Actions</strong> - Workflows and third-party Actions</summary>

- Are Actions pinned to full commit SHAs (not tags)?
- Are workflow permissions set to least privilege?
- Are `pull_request_target` triggers used safely?
- Is CODEOWNERS configured for workflow files?
</details>

<details>
<summary><strong>Infrastructure as Code</strong> - Terraform modules and providers</summary>

- Are modules and providers version-pinned?
- Is the lockfile committed?
- Are dangerous provisioners (local-exec, remote-exec) flagged?
- Is state encrypted and access-controlled?
</details>

<details>
<summary><strong>AI/ML Models</strong> - Model files and loading code</summary>

- Are pickle files present (arbitrary code execution risk)?
- Does code use unsafe `torch.load()` or `pickle.load()`?
- Are models verified by hash or signature?
- Are safer serialization formats (safetensors, ONNX) used?
</details>

<details>
<summary><strong>IDE Extensions</strong> - VS Code, devcontainers</summary>

- Are recommended extensions from verified publishers?
- Are secrets accidentally stored in workspace settings?
- Is devcontainer extension isolation configured?
- Is an extension allowlist in place?
</details>

<details>
<summary><strong>Credentials</strong> - Secrets, tokens, API keys</summary>

- Are there hardcoded secrets in source code or config files?
- Are `.env` files properly gitignored?
- Are secrets present in git history?
- Is a pre-commit hook set up to prevent future leaks?
- Are long-lived cloud credentials replaceable with OIDC?
- Are package manager auth tokens secured?
- Are commits and tags signed to prevent impersonation?
</details>

## The Skill

Beyond the slash commands, this plugin includes a `supply-chain-hardening` skill that activates automatically when you ask Claude about supply chain security topics. It gives Claude access to:

- A prioritized checklist of "Do Right Now" vs "Do Eventually" items
- A catalog of open-source and paid security tools for each domain
- Ready-to-use configuration snippets for all supported package managers and tools

Just ask Claude something like *"How do I prevent dependency confusion attacks?"* and it will pull from this knowledge automatically.

## Status and Contributing

This plugin is a new initiative and actively evolving. Coverage, scanner integrations, and fix strategies will improve over time. If you run into issues, have ideas, or want to improve the checks:

- **Open an issue**: [github.com/latiotech/secure-supply-chain-skills/issues](https://github.com/latiotech/secure-supply-chain-skills/issues)
- **Submit a PR**: See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines

Feedback from real-world usage is especially valuable — if a check produces a false positive, misses something, or breaks your workflow, let us know.