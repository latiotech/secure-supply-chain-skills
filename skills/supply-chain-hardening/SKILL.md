---
name: supply-chain-hardening
description: >
  Use this skill when the user asks to "harden supply chain", "secure dependencies",
  "pin versions", "audit packages", "secure GitHub Actions", "harden containers",
  "secure IaC", "audit extensions", "secure AI models", "audit pickle files",
  "scan for secrets", "harden credentials", "rotate tokens", "audit credentials", or
  needs guidance on software supply chain security, dependency management,
  credential hygiene, secret scanning, or preventing supply chain attacks. Also
  trigger when the user mentions tools like Grype, Cosign, Sigstore, Checkov,
  Hadolint, Zizmor, Picklescan, ModelScan, SafeTensors, Betterleaks, Gitleaks,
  TruffleHog, or references SLSA.
version: 0.2.0
---

# Supply Chain Security Hardening

Secure the software supply chain across seven domains: third-party packages, container images, GitHub Actions, infrastructure-as-code, AI/ML models, IDE extensions, and credentials/secrets.

## How to Use This Skill

This skill provides two modes of operation:

### Action Commands - Take Realistic Action by Default

These commands scan the codebase, make changes directly (pin hashes, fix configs, resolve SHAs), and explain each change. They are the default way to harden a project.

- `/audit-supply-chain` - Full audit with auto-fixes for critical items
- `/harden-packages` - Pin versions, disable scripts, secure registries
- `/harden-containers` - Pin digests, enforce non-root, add .dockerignore
- `/harden-actions` - Pin SHAs, set permissions, fix injection, add Dependabot
- `/harden-iac` - Pin modules/providers, generate lockfiles, flag provisioners
- `/harden-ai-ml` - Fix unsafe deserialization, pin model sources
- `/harden-ide-extensions` - Audit extensions, remove secrets, add devcontainer
- `/harden-credentials` - Scan for leaked secrets, set up pre-commit hooks, harden credential hygiene
- `/audit-credentials` - Find long-lived tokens, hardcoded secrets, credentials to rotate or replace

### Walkthrough Commands - Guided Setup for Advanced Items

These commands are interactive walkthroughs for configurations that require steps outside the codebase (cloud provider setup, Kubernetes config, GitHub settings). They should be run separately.

- `/setup-oidc` - Replace cloud credentials with OIDC tokens
- `/setup-image-signing` - Sign images with Cosign/Sigstore
- `/setup-tag-rulesets` - Protect tags from force-push attacks
- `/setup-admission-control` - Enforce image policies in Kubernetes
- `/setup-sbom` - Generate SBOMs and provenance attestations
- `/setup-runner-monitoring` - Add runtime detection to CI runners

## Reference Material

1. **Read the checklist** at `references/checklist.md` to understand what needs to be done
2. **Read the tool reference** at `references/tools.md` for open source and paid options
3. **Read the implementation configs** at `references/package-configs.md` for copy-paste configurations
4. **Read the container guidance** at `references/container-configs.md` for Dockerfile and image hardening
5. **Read the actions guidance** at `references/actions-configs.md` for GitHub Actions hardening
6. **Read the iac guidance** at `references/iac-configs.md` for Terraform/IaC hardening
7. **Read the credential guidance** at `references/credentials-configs.md` for secret scanning and credential hygiene
8. **Read the AI/ML guidance** at `references/ai-ml-configs.md` for model security configurations

## Core Principles

1. **Pin everything.** Versions, digests, commit SHAs. Floating references are attack surface.
2. **Never fabricate pins.** Always resolve versions, SHAs, and digests from live sources (lockfiles, `git ls-remote`, `docker inspect`, registry APIs). If resolution fails, add a `# TODO: pin` comment with the exact command the user should run - never guess a value.
3. **Least privilege everywhere.** Tokens, permissions, workflow scopes, container users.
4. **Verify provenance.** Sign artifacts, check signatures, require attestations.
5. **Detect at runtime.** Scanning at build time isn't enough - monitor what actually runs.
6. **Plan for cascading compromise.** One stolen token can hit npm, PyPI, Docker Hub, and extension marketplaces simultaneously.

## Domain Quick Reference

### Third-Party Packages
Pin versions. Require cooldown periods. Disable install scripts. Use a registry proxy/firewall. Scan for malware. Generate SBOMs. See `references/package-configs.md` for language-specific configurations.

### Container Images
Pin by digest. Run as non-root. Use minimal base images. Sign with Cosign. Scan continuously. Use admission controllers. See `references/container-configs.md`.

### GitHub Actions
Pin to commit SHAs. Set explicit permissions. Audit for `pull_request_target`. Enable tag protection rules. Monitor runner activity. See `references/actions-configs.md`.

### Infrastructure-as-Code
Pin module versions. Scan with Checkov/tflint. Enforce policy with OPA/Sentinel. Lock down provisioners. Require signed commits. See `references/iac-configs.md`.

### AI/ML Models
Never load untrusted pickles. Use SafeTensors/ONNX. Verify model hashes. Scan with Picklescan/ModelScan. Sandbox model loading.

### IDE Extensions
Audit installed extensions. Use osquery for fleet visibility. Enforce allowlists. Use VS Code Profiles. Run dev environments in containers.

### Credentials & Secrets
Scan for leaked secrets with Betterleaks. Set up pre-commit hooks to prevent future leaks. Harden .gitignore. Rotate compromised credentials. Replace long-lived tokens with OIDC where possible. See `references/credentials-configs.md`.

## When Making Changes

Action commands follow these principles:

1. **Make changes by default** - pin versions, resolve SHAs, fix configs directly
2. **Explain each change** as it's made - what was changed and why
3. **Don't break existing functionality** - test after changes
4. **Prioritize "Do Right Now" items** from the checklist before "Do Eventually" items
5. **Flag items that need manual attention** separately from automated fixes
6. **Point users to walkthrough commands** for advanced items that require external configuration
