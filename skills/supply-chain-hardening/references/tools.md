# Supply Chain Security Tools Reference

Companion to the [Supply Chain Security Checklist](checklist.md). Open source and paid options for each subcategory.

---

## 📦 Third-Party Packages

### Package Scanners

| Tool | Type | What It Does |
|------|------|-------------|
| [Aikido SafeChain](https://github.com/AikidoSec/safe-chain) | OSS | Malware scanner wrapper for npm/yarn/pnpm; blocks malicious packages in real-time |
| [GuardDog](https://github.com/DataDog/guarddog) | OSS | Identifies malicious PyPI and npm packages using Semgrep + metadata heuristics |
| [Socket.dev](https://socket.dev/) | Freemium | Deep package inspection detecting supply chain attacks before they hit your code |
| [Phylum](https://www.phylum.io/) | Paid | SaaS scanner analyzing packages for malicious code, vulns, and author risk |
| [Semgrep Supply Chain](https://semgrep.dev/products/semgrep-supply-chain/) | Freemium | Reachability-aware SCA - only flags vulns in code paths you actually call |
| [Snyk Open Source](https://snyk.io/product/open-source-security-management/) | Freemium | Dependency vulnerability scanner with fix PRs; free tier for public repos |
| [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) | OSS | Detects publicly disclosed vulnerabilities in project dependencies |

### Package Registries & Proxies

| Tool | Type | What It Does |
|------|------|-------------|
| [Artifactory](https://jfrog.com/artifactory/) | Paid (OSS edition available) | Universal artifact repository with proxy caching and access control |
| [Nexus Repository](https://www.sonatype.com/products/sonatype-nexus-repository) | Paid (Community edition free) | Artifact repository manager; the other major Artifactory competitor |
| [Cloudsmith](https://cloudsmith.com/) | Freemium | Cloud-native artifact management; free for open source projects |
| [Verdaccio](https://github.com/verdaccio/verdaccio) | OSS | Lightweight Node.js private npm proxy registry you can self-host |
| [GitHub Packages](https://docs.github.com/en/packages) | Freemium | GitHub-integrated registry; free for public packages |
| [Chainguard Libraries](https://www.chainguard.dev/libraries) | Paid | Source-built, SLSA L3 package registry for Java/Python/JS; blocks 98%+ of malware by only including packages with verified, buildable source |
| [Socket Firewall](https://docs.socket.dev/docs/socket-firewall-overview) | Freemium | Intelligent proxy between your package manager and registries; intercepts and blocks malicious packages before install across npm, pip, cargo |
| [Sonatype Repository Firewall](https://www.sonatype.com/products/sonatype-repository-firewall) | Paid | AI-powered proxy that quarantines suspicious components before they enter your artifact repository; works with Nexus, Artifactory, or standalone |
| [Root Library Catalog](https://www.root.io/) | Paid | Automated vulnerability remediation for packages and container images; resolves deps through pkg.root.io with patching SLAs |
| [Minimus Supply Chain Protection](https://www.minimus.io/) | Paid | Controls on packages by origin, maturity, and usage; also offers 1,200+ hardened container images and SBOM generation |

### SBOM Generation

| Tool | Type | What It Does |
|------|------|-------------|
| [Syft](https://github.com/anchore/syft) | OSS | Generates SBOMs from container images and filesystems |
| [cdxgen](https://github.com/CycloneDX/cdxgen) | OSS | Polyglot SBOM generator supporting 90+ package managers |
| [Trivy](https://github.com/aquasecurity/trivy) | OSS | Multi-scanner that also generates SBOMs in CycloneDX/SPDX formats |
| [Microsoft SBOM Tool](https://github.com/microsoft/sbom-tool) | OSS | Microsoft's general-purpose SBOM generation tool |

### Provenance & Signing

| Tool | Type | What It Does |
|------|------|-------------|
| [Sigstore](https://docs.sigstore.dev/) | OSS | Keyless signing and verification for software artifacts (CNCF) |
| [npm provenance](https://docs.npmjs.com/generating-provenance-statements/) | Built-in | Sigstore-powered SLSA provenance for npm packages via GitHub Actions |
| [in-toto](https://in-toto.io/) | OSS | Framework for verifying the integrity of each step in a software supply chain |

---

## 🐳 Container Images

### Image Scanners

| Tool | Type | What It Does |
|------|------|-------------|
| [Grype](https://github.com/anchore/grype) | OSS | Fast container and SBOM vulnerability scanner (Anchore) |
| [Trivy](https://github.com/aquasecurity/trivy) | OSS | Swiss army knife - vulns, misconfigs, secrets, SBOMs in one tool |
| [Clair](https://github.com/quay/clair) | OSS | Layer-by-layer container vulnerability analyzer (Red Hat/Quay) |
| [Docker Scout](https://docs.docker.com/scout/) | Freemium | Docker's native image scanning; basic free tier |
| [Snyk Container](https://docs.snyk.io/scan-with-snyk/snyk-container) | Freemium | Container scanning with base image upgrade recommendations |

### Image Signing & Verification

| Tool | Type | What It Does |
|------|------|-------------|
| [Cosign](https://github.com/sigstore/cosign) | OSS | Keyless container image signing and verification (Sigstore) |
| [Notation](https://github.com/notaryproject/notation) | OSS | OCI-native signing tool (Microsoft-backed, successor to Notary v2) |
| [Docker Content Trust](https://docs.docker.com/engine/security/trust/) | Built-in | Legacy DCT signing; being phased in favor of Sigstore |

### Admission Controllers

| Tool | Type | What It Does |
|------|------|-------------|
| [Kyverno](https://kyverno.io/) | OSS | Kubernetes-native policy engine; YAML-based, no Rego needed (CNCF) |
| [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper) | OSS | Policy controller using Open Policy Agent and Rego (CNCF) |
| [Kubewarden](https://github.com/kubewarden/kubewarden) | OSS | WebAssembly-based policy engine; write policies in any Wasm-compatible language |

### Container Registries

| Tool | Type | What It Does |
|------|------|-------------|
| [Harbor](https://goharbor.io/) | OSS | Enterprise registry with scanning, replication, RBAC (CNCF Graduated) |
| [Docker Hub](https://hub.docker.com) | Freemium | Largest public registry; paid plans for private repos and teams |
| [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) | Freemium | GitHub-integrated; free for public images |
| [Amazon ECR](https://aws.amazon.com/ecr/) | Paid | AWS managed registry with scanning built in |
| [Google Artifact Registry](https://cloud.google.com/artifact-registry) | Paid | GCP multi-format artifact storage |
| [Azure Container Registry](https://azure.microsoft.com/en-us/products/container-registry) | Paid | Azure managed registry with geo-replication |
| [Quay.io](https://quay.io/) | Freemium | Red Hat's container registry with security scanning |

### Minimal Base Images

| Tool | Type | What It Does |
|------|------|-------------|
| [Alpine](https://hub.docker.com/_/alpine) | OSS | ~5MB minimal Linux distro; musl-based |
| [Distroless](https://github.com/GoogleContainerTools/distroless) | OSS | Google's minimal images - no shell, no package manager, just your app |
| [Wolfi](https://github.com/wolfi-dev) | OSS | Chainguard's security-focused undistro; glibc-based (better compatibility than Alpine) |
| [Chainguard Images](https://images.chainguard.dev/) | Freemium | Hardened distroless images rebuilt daily with SBOMs; 1,300+ images |

### Dockerfile Linting

| Tool | Type | What It Does |
|------|------|-------------|
| [Hadolint](https://github.com/hadolint/hadolint) | OSS | Dockerfile linter with integrated ShellCheck for bash validation |
| [Dockle](https://github.com/goodwithtech/dockle) | OSS | Container image linter focused on CIS benchmark best practices |

---

## ⚙️ GitHub Actions

### Actions Security & Analysis

| Tool | Type | What It Does |
|------|------|-------------|
| [Zizmor](https://github.com/zizmorcore/zizmor) | OSS | Static analysis for GitHub Actions security misconfigurations |
| [StepSecurity Harden Runner](https://github.com/step-security/harden-runner) | Freemium | Monitors network egress and filesystem activity in CI runners |
| [pin-github-action](https://app.stepsecurity.io/secureworkflow) | OSS | Pins Actions to commit SHAs automatically |
| [OpenSSF Scorecard](https://scorecard.dev/) | OSS | Automated security scoring across 18 checks for open source projects |
| [actionlint](https://github.com/rhysd/actionlint) | OSS | Linter for GitHub Actions workflow files; catches syntax and type errors |

### Dependency Update Automation

| Tool | Type | What It Does |
|------|------|-------------|
| [Dependabot](https://docs.github.com/en/code-security/dependabot) | Built-in | GitHub-native; auto-creates PRs for outdated dependencies and Actions |
| [Renovate](https://github.com/renovatebot/renovate) | OSS (AGPL) | Cross-platform dependency updater; 90+ package managers, highly configurable |

---

## 🏗️ Infrastructure-as-Code

### IaC Scanners

| Tool | Type | What It Does |
|------|------|-------------|
| [Checkov](https://github.com/bridgecrewio/checkov) | OSS | Scans Terraform, CloudFormation, K8s, and more for misconfigurations |
| [tflint](https://github.com/terraform-linters/tflint) | OSS | Terraform-specific linter catching errors, deprecations, and naming issues |
| [Trivy](https://github.com/aquasecurity/trivy) | OSS | IaC misconfiguration scanning alongside everything else Trivy does |
| [KICS](https://github.com/Checkmarx/kics) | OSS | Checkmarx's IaC scanner; 2,400+ out-of-the-box queries |
| [Terrascan](https://github.com/tenable/terrascan) | OSS | OPA-powered IaC analyzer with 500+ built-in policies (Tenable) |

### Policy Engines

| Tool | Type | What It Does |
|------|------|-------------|
| [OPA](https://www.openpolicyagent.org/) | OSS | General-purpose policy engine using Rego (CNCF Graduated) |
| [HashiCorp Sentinel](https://developer.hashicorp.com/sentinel) | Paid | Policy-as-code embedded in Terraform Cloud; requires governance tier |
| [Styra DAS](https://www.styra.com/) | Paid | Enterprise OPA management - GUI, decision logging, policy distribution |

### IaC Platforms

| Tool | Type | What It Does |
|------|------|-------------|
| [Atlantis](https://github.com/runatlantis/atlantis) | OSS | Self-hosted GitOps for Terraform with PR-based plan/apply |
| [OpenTofu](https://opentofu.org/) | OSS | Community fork of Terraform (Linux Foundation) |
| [Crossplane](https://github.com/crossplane/crossplane) | OSS | Kubernetes-native infrastructure control plane (CNCF) |
| [Terraform Cloud](https://developer.hashicorp.com/terraform/cloud-docs) | Paid | HashiCorp's managed Terraform service with remote state and governance |
| [Spacelift](https://spacelift.io/) | Paid | IaC orchestration with built-in OPA, drift detection, and policy |
| [env0](https://www.env0.com/) | Paid | IaC automation with cost management; Terraform, OpenTofu, Pulumi |
| [Firefly](https://www.firefly.ai/) | Paid | Infrastructure asset management and drift detection |
| [Pulumi](https://github.com/pulumi/pulumi) | Hybrid | IaC in real programming languages; OSS SDK + paid cloud service |

---

## 🤖 AI/ML Models

### Model Scanners

| Tool | Type | What It Does |
|------|------|-------------|
| [Picklescan](https://github.com/mmaitre314/picklescan) | OSS | Detects suspicious operations in Python pickle files |
| [ModelScan](https://github.com/protectai/modelscan) | OSS | Scans serialized ML models (H5, pickle, SavedModel) for attacks |
| [Fickling](https://github.com/trailofbits/fickling) | OSS | Trail of Bits' pickle decompiler and static analyzer |

### Safe Serialization

| Tool | Type | What It Does |
|------|------|-------------|
| [SafeTensors](https://github.com/huggingface/safetensors) | OSS | Secure tensor format that avoids arbitrary code execution (Hugging Face) |
| [ONNX](https://onnx.ai/) | OSS | Cross-framework model interchange format; no code execution risk |

### AI Security Platforms

| Tool | Type | What It Does |
|------|------|-------------|
| [HiddenLayer](https://www.hiddenlayer.com/) | Paid | Model supply chain security, attack simulation, runtime detection |
| [Prisma AIRS](https://www.paloaltonetworks.com/prisma/prisma-ai-runtime-security) | Paid | Palo Alto's AI security covering model, data, and agent security |
| [Robust Intelligence (Cisco)](https://www.cisco.com/site/us/en/products/security/ai-defense/robust-intelligence-is-part-of-cisco/index.html) | Paid | AI governance and model testing; acquired by Cisco in 2024 |

---

## 🧩 IDE Extensions & Developer Tools

### Extension Auditing & Management

| Tool | Type | What It Does |
|------|------|-------------|
| [osquery](https://github.com/osquery/osquery) | OSS | SQL-based endpoint visibility; `vscode_extensions` table for fleet-wide auditing |
| [Santa](https://santa.dev/) | OSS | Binary authorization system for macOS; can audit and control what runs on developer machines |
| [VS Code Profiles](https://code.visualstudio.com/docs/configure/profiles) | Built-in | Standardize extensions per workspace or team |
| VS Code enterprise policies | Built-in | Restrict Marketplace access and enforce extension allowlists via GPO/MDM |

### Remote/Ephemeral Dev Environments

| Tool | Type | What It Does |
|------|------|-------------|
| [Dev Containers](https://containers.dev/) | OSS | Open spec for containerized development environments |
| [GitHub Codespaces](https://github.com/features/codespaces) | Paid | GitHub's cloud dev environments; free tier for individual devs |
| [Gitpod](https://gitpod.io/) | Freemium | Cloud IDE with free tier; self-hosted option available |
| [DevPod](https://devpod.sh/) | OSS | Open source Codespaces alternative; runs on any infrastructure |
| [Coder](https://coder.com/) | Hybrid | Self-hosted remote dev platform; OSS core + enterprise features |

---

## 🔑 Credentials & Secrets

### Commit & Tag Signing

| Tool | Type | What It Does |
|------|------|-------------|
| [Git SSH signing](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification) | Built-in | Sign commits with existing SSH keys (Git 2.34+); easiest setup for most teams |
| [GPG signing](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits) | Built-in | Traditional commit signing with GPG keys; widest platform support |
| [gitsign](https://github.com/sigstore/gitsign) | OSS | Keyless commit signing using OIDC identity (Sigstore); no keys to manage |

### Secret Scanning

| Tool | Type | What It Does |
|------|------|-------------|
| [GitHub Secret Scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning) | Built-in | Detects leaked secrets in repos; push protection blocks commits with secrets |
| [Betterleaks](https://github.com/betterleaks/betterleaks) | OSS | **Recommended.** Fast, accurate secret scanner with low false-positive rate; supports staged-only and history scanning |

### Secrets Managers

| Tool | Type | What It Does |
|------|------|-------------|
| [HashiCorp Vault](https://www.vaultproject.io/) | Hybrid | Dynamic secret generation, encryption as a service, lease-based credentials |
| [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) | Paid | AWS-native; auto-rotates RDS, Redshift, DocumentDB credentials |
| [1Password Developer](https://developer.1password.com/) | Paid | Secret references in CI/CD; CLI and SDKs for secret injection |
| [Infisical](https://github.com/Infisical/infisical) | Hybrid | Open source secrets manager with agent injection and rotation |
