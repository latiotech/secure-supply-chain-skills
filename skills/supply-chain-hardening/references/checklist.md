# Supply Chain Security Checklist

---

## Third-Party Packages

### Do Right Now

- [ ] **Pin all dependencies to specific versions** → [Pnpm Security Docs](https://pnpm.io/supply-chain-security) (applies to other languages and their package managers too)
- [ ] **Require a package cooldown period** on your builds (don't auto-pull brand-new packages) → [Renovate `minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage) · [Aikido SafeChain](https://github.com/AikidoSec/safe-chain) (JS/Python) · [Socket Firewall](https://docs.socket.dev/docs/socket-firewall-overview)
- [ ] **Disable pre-install and post-install scripts** in your package manager → [Pnpm Secure Install](https://pnpm.io/supply-chain-security)

### Do Eventually

- [ ] **Review every new dependency addition** in PRs before merging → [Snyk Advisor](https://snyk.io/advisor/)
- [ ] **Deploy a malware scanner** locally and in CI/CD that blocks on detection → [Aikido SafeChain](https://github.com/AikidoSec/safe-chain) · [GuardDog](https://github.com/DataDog/guarddog)
- [ ] **Enable namespace/scope restrictions** to prevent dependency confusion → [Artifactory](https://docs.jfrog.com/artifactory/docs/jfrog-artifactory) · [Cloudsmith](https://help.cloudsmith.io/docs/create-a-repository)
- [ ] **Enforce allow-lists** for approved packages
- [ ] **Verify package provenance** with Sigstore → [Sigstore](https://docs.sigstore.dev/about/overview/)
- [ ] **Generate SBOMs** for your dependency tree → [CycloneDX](https://cyclonedx.org/getting-started/) · [Syft](https://github.com/anchore/syft)
- [ ] **Run packages in sandboxed install environments** to detect install-time behavior → [StepSecurity](https://docs.stepsecurity.io/harden-runner), or other eBPF/CADR sensors

---

## Container Images

### Do Right Now

- [ ] **Audit your images** - ensure you're using official or verified base images only → [Docker Scout](https://docs.docker.com/scout/), [Grype](https://github.com/anchore/grype)
- [ ] **Pin base images by digest**, not just tag (`FROM node:18@sha256:...`)
- [ ] **Stop running containers as root** → [Hadolint](https://hadolint.github.io/hadolint/)
- [ ] **Scan images for known vulnerabilities** → [Grype](https://github.com/anchore/grype) · [Docker Scout](https://docs.docker.com/scout/)

### Do Eventually

- [ ] **Implement image signing and verification** to prevent tampering → [Cosign](https://docs.sigstore.dev/cosign/)
- [ ] **Set up admission controllers** to block unsigned or unscanned images → [Kyverno](https://kyverno.io/)
- [ ] **Scan images continuously in the registry**, not just at build time → [Harbor](https://goharbor.io/) · [Dockle](https://github.com/goodwithtech/dockle)
- [ ] **Automate weekly rebuilds** of base images
- [ ] **Switch to minimal base images**, and/or use multi-step builds → [Alpine](https://hub.docker.com/_/alpine) · [Distroless](https://github.com/GoogleContainerTools/distroless)
- [ ] **Build images in hermetic, reproducible environments**
- [ ] **Generate and attach SBOMs and SLSA provenance attestations** to every image → [SLSA](https://slsa.dev/) · [in-toto](https://in-toto.io/)

---

## GitHub Actions

### Do Right Now

- [ ] **Pin all Actions to full commit SHAs** (not tags or branches) → [pin-github-action](https://app.stepsecurity.io/secureworkflow)
- [ ] **Set `permissions:` explicitly** in every workflow to least-privilege
- [ ] **Use CODEOWNERS** to require review on `.github/workflows/` changes
- [ ] **Audit which third-party Actions** you're currently using, and scan them for issues like `pull_request_target` triggers → [Zizmor](https://github.com/zizmorcore/zizmor)
- [ ] **Enable tag protection rules** - create a ruleset to prevent forced tag releases
- [ ] **Replace classic PATs with fine-grained tokens** scoped to specific repos with a maximum expiration → [GitHub Docs: PAT Policies](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization)

### Do Eventually

- [ ] **Enable GitHub's Action allow-list** to restrict which Actions can run
- [ ] **Monitor network egress and filesystem activity** in workflows → [StepSecurity Harden Runner](https://docs.stepsecurity.io/harden-runner), or other eBPF sensor
- [ ] **Use OpenSSF Scorecard** to evaluate Action trustworthiness → [OpenSSF Scorecard](https://scorecard.dev/)
- [ ] **Implement OIDC for cloud deployments** instead of long-lived secrets
- [ ] **Separate CI and CD workflows** with environment-based approvals
- [ ] **Enforce SHA pinning at the org level** via GitHub policy
- [ ] **Run self-hosted runners in ephemeral, isolated environments**
- [ ] **Build internal Action mirrors** for critical dependencies
- [ ] **Set up automated Action version updates** → [Dependabot](https://docs.github.com/en/code-security/tutorials/secure-your-dependencies/dependabot-quickstart-guide) · [Renovate](https://docs.renovatebot.com/)

---

## Infrastructure-as-Code Modules

### Do Right Now

- [ ] **Pin Terraform modules and providers** to exact versions
- [ ] **Use only modules from the official verified registry** or your org's private registry
- [ ] **Run IaC scanning** for misconfigurations → [Checkov](https://www.checkov.io/) · [tflint](https://github.com/terraform-linters/tflint)

### Do Eventually

- [ ] **Implement Terraform state encryption** and access controls
- [ ] **Use OPA/Sentinel policies** to enforce guardrails (no public S3 buckets, no wildcard IAM, etc.) → [Sentinel](https://developer.hashicorp.com/sentinel/docs/concepts/language) · [OPA](https://www.openpolicyagent.org/)
- [ ] **Enable drift detection** to catch out-of-band changes → [Spacelift](https://spacelift.io/) · [env0](https://www.env0.com/)
- [ ] **Lock down local-exec and external provisioners**
- [ ] **Run all IaC applies through audited, ephemeral CI runners** → [Atlantis](https://www.runatlantis.io/) · [Terraform Cloud](https://developer.hashicorp.com/terraform/cloud-docs)
- [ ] **Require signed commits** for IaC repos → see `/setup-commit-signing`
- [ ] **Build internal module libraries** with security-reviewed defaults
- [ ] **Continuously reconcile running infrastructure** against declared state → [Crossplane](https://www.crossplane.io/) · [Firefly](https://www.firefly.ai/)

---

## AI/ML Models

### Do Right Now

- [ ] **Never load untrusted pickle files**
- [ ] **Prefer safer serialization formats** over pickle/PyTorch native → [SafeTensors](https://github.com/huggingface/safetensors) · [ONNX](https://onnx.ai/)
- [ ] **Only pull models from verified organizations** on Hugging Face → [Hugging Face Hub](https://huggingface.co/docs/hub/index)
- [ ] **Document which models your team uses** and where they come from

### Do Eventually

- [ ] **Run Picklescan on all PyTorch models** before loading → [Picklescan](https://github.com/mmaitre314/picklescan) · [ModelScan](https://github.com/protectai/modelscan)
- [ ] **Host models in a private registry** with access controls
- [ ] **Verify model hashes** against known-good checksums
- [ ] **Implement model cards** and provenance documentation
- [ ] **Run all model loading in sandboxed environments** with no network access
- [ ] **Implement model signing and verification**
- [ ] **Conduct adversarial testing** to detect poisoned models → [HiddenLayer](https://www.hiddenlayer.com/) · [Prisma AIRS](https://www.paloaltonetworks.com/prisma/prisma-ai-runtime-security)
- [ ] **Maintain an internal model registry** with security review gates

---

## IDE Extensions & Developer Tools

### Do Right Now

- [ ] **Audit currently installed extensions** across your team → [VS Code Marketplace](https://marketplace.visualstudio.com/vscode) · [Santa](https://santa.dev/) · [OSQuery](https://osquery.io/)
- [ ] **Only install extensions from verified publishers** with meaningful install counts → [Wiz: VS Code Supply Chain Risk](https://www.wiz.io/blog/supply-chain-risk-in-vscode-extension-marketplaces)
- [ ] **Remove unused extensions**
- [ ] **Review extension permissions before installing** (network access, filesystem access)
- [ ] **Give developer training** on extension usage and risks

### Do Eventually

- [ ] **Publish an org-approved extension allowlist**
- [ ] **Use VS Code Profiles** to standardize extensions across teams → [VS Code Profiles](https://code.visualstudio.com/docs/configure/profiles)
- [ ] **Restrict VS Code Marketplace access** via enterprise policy
- [ ] **Have a plan to rotate secrets** that live on developer workstations
- [ ] **Run dev environments in remote containers** or cloud-based environments → [GitHub Codespaces](https://docs.github.com/en/codespaces/overview) · [Gitpod](https://gitpod.io/) · [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers)
- [ ] **Implement network segmentation** for developer workstations
- [ ] **Monitor extension behavior** at the endpoint level
- [ ] **Use ephemeral dev environments** that reset between sessions

---

## Credentials & Secrets

### Do Right Now

- [ ] **Remove hardcoded credentials from code** - scan for API keys, private keys, connection strings, and tokens in source files → [Betterleaks](https://github.com/betterleaks/betterleaks)
- [ ] **Check git history for leaked secrets** - previously committed secrets are still exposed even if deleted from the working tree
- [ ] **Ensure `.env` files are gitignored** - never commit environment files with credentials
- [ ] **Use fine-grained PATs** instead of classic tokens - they support repo-level scoping and mandatory expiration → [GitHub Docs: Fine-grained PATs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [ ] **Prefer `GITHUB_TOKEN`** over PATs in workflows - it's scoped to the current repo and expires after the job

### Do Eventually

- [ ] **Enable commit and tag signing** - prevents commit impersonation and ensures code provenance → [GitHub Docs: Signing commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits) · [gitsign](https://github.com/sigstore/gitsign)
- [ ] **Require signed commits on protected branches** - enforce via branch protection rules or repository rulesets
- [ ] **Use a secrets manager** (Vault, AWS Secrets Manager, 1Password, etc.) instead of environment variables for sensitive values
- [ ] **Prefer GitHub Apps over PATs** for automation - apps have granular permissions, audit logs, and installation-scoped tokens
- [ ] **Have a credential rotation plan** - when one token leaks, assume lateral movement to npm, PyPI, Docker Hub, and extension marketplaces; plan for mass rotation
- [ ] **Audit GitHub App installations** - flag apps with `repository_selection: "all"` or excessive permissions
- [ ] **Consider pre-commit hooks for secret detection** - optional extra layer to block secrets before they reach git history → [Betterleaks](https://github.com/betterleaks/betterleaks) · [pre-commit](https://pre-commit.com/)
- [ ] **Enable GitHub secret scanning push protection** - blocks pushes containing detected secret patterns
