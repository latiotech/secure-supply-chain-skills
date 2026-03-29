---
description: "Walkthrough: Set up container image signing with Cosign/Sigstore"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, docker:*, cosign:*, gh:*)
---

Interactive walkthrough to set up container image signing and verification using Cosign (part of the Sigstore project). This ensures images haven't been tampered with between build and deploy.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/container-configs.md` for Cosign reference configs.

## Why sign images?

Without image signing, anyone with push access to your registry can replace an image. Signing creates a cryptographic proof that the image was built by your CI/CD pipeline and hasn't been modified since.

## Step 1: Assess current state

Check the project for:
- Existing Dockerfiles and image build workflows
- Current registry (Docker Hub, GHCR, ECR, GCR, ACR, Harbor)
- Whether images are currently tagged or digest-referenced
- Whether any signing is already in place

Present findings to the user.

## Step 2: Choose signing approach

Explain the two options:

### Keyless signing (recommended)
- Uses OIDC identity (GitHub Actions, Google, etc.) - no keys to manage
- Signatures are logged in Rekor (Sigstore transparency log) for auditability
- Best for CI/CD pipelines

### Key-based signing
- Uses a static key pair - you manage the private key
- Better for air-gapped or highly regulated environments
- Requires secure key storage (KMS, Vault, etc.)

Ask the user which approach fits their setup.

## Step 3: Set up keyless signing in CI

Walk the user through adding Cosign signing to their image build workflow:

1. **Add Cosign install step**
   ```yaml
   - name: Install Cosign
     uses: sigstore/cosign-installer@dc72c7d5c4d10cd6bcb8cf6e3fd625a9e5e537da # v3.7.0
   ```

2. **Add OIDC permissions**
   ```yaml
   permissions:
     id-token: write    # for signing
     packages: write    # for pushing signature to registry
     contents: read
   ```

3. **Sign after push**
   ```yaml
   - name: Sign image
     run: cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE }}@${{ steps.build.outputs.digest }}
   ```

4. **Attach SBOM (optional but recommended)**
   ```yaml
   - name: Generate and attach SBOM
     run: |
       trivy image --format cyclonedx --output sbom.json ${{ env.IMAGE }}@${{ steps.build.outputs.digest }}
       cosign attach sbom --sbom sbom.json ${{ env.REGISTRY }}/${{ env.IMAGE }}@${{ steps.build.outputs.digest }}
   ```

Make the actual workflow file changes for the user, explaining each addition.

## Step 4: Set up verification

Walk the user through verifying signed images:

1. **Local verification**
   ```bash
   cosign verify $REGISTRY/$IMAGE@sha256:... \
     --certificate-identity=https://github.com/OWNER/REPO/.github/workflows/build.yml@refs/heads/main \
     --certificate-oidc-issuer=https://token.actions.githubusercontent.com
   ```

2. **In deployment scripts** - add a verification step before deploying

3. **In Kubernetes** - suggest running `/setup-admission-control` to enforce verification at the cluster level

## Step 5: Verify the setup

After making changes:
- Suggest pushing a branch and running the build workflow
- Show how to check that the signature was created: `cosign tree $REGISTRY/$IMAGE@sha256:...`
- Explain what the Rekor transparency log entry means

## Next steps

- Run `/setup-admission-control` to enforce that only signed images can run in your cluster
- Run `/setup-sbom` to add full SBOM and provenance attestation pipelines
