---
description: "Walkthrough: Set up SBOM generation and provenance attestation pipelines"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, syft:*, trivy:*, cosign:*, npm:*, pip:*)
---

Interactive walkthrough to set up Software Bill of Materials (SBOM) generation and SLSA provenance attestations in your CI/CD pipeline. SBOMs provide a complete inventory of what's in your software; provenance attestations prove where and how it was built.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/tools.md` for SBOM tool options.

## Why SBOMs and provenance?

- **SBOMs** let you instantly check if your software contains a newly disclosed vulnerable component (instead of grepping through code)
- **Provenance attestations** prove your artifacts were built by your CI/CD pipeline on specific source code - not hand-modified or injected
- Both are increasingly required by regulation (Executive Order 14028, EU CRA) and enterprise customers

## Step 1: Assess current state

Check the project for:
- What artifacts are produced (npm packages, Docker images, Go binaries, Python wheels, etc.)
- Existing CI/CD workflows that build and publish artifacts
- Whether any SBOM tools are already configured (Syft, Trivy, cdxgen, Microsoft SBOM Tool)
- Whether image signing is set up (SBOMs can be attached to signed images)

Present findings and ask:
- What format do you prefer? **CycloneDX** (more adoption in security tools) or **SPDX** (more adoption in compliance/legal)
- Do you need SBOMs for packages, containers, or both?

## Step 2: Choose SBOM tools

Explain the options:

### For container images
- **Syft** - Anchore's dedicated SBOM generator. Excellent format support.
- **Trivy** - generates SBOMs alongside vulnerability scanning. Good if you already use Trivy.
- Both output CycloneDX and SPDX.

### For source code / packages
- **cdxgen** - polyglot, supports 90+ package managers
- **npm provenance** - built-in npm feature for publishing with Sigstore provenance
- Language-specific tools (cargo-sbom, pip-sbom, etc.)

Recommend the best fit based on what the project produces.

## Step 3: Add SBOM generation to CI

Walk the user through adding SBOM generation to their build workflow.

### For container images

```yaml
- name: Generate SBOM
  run: |
    # Using Syft
    syft ${{ env.IMAGE }}@${{ steps.build.outputs.digest }} -o cyclonedx-json > sbom.cdx.json

    # Or using Trivy
    trivy image --format cyclonedx --output sbom.cdx.json ${{ env.IMAGE }}@${{ steps.build.outputs.digest }}

- name: Attach SBOM to image
  run: cosign attach sbom --sbom sbom.cdx.json ${{ env.REGISTRY }}/${{ env.IMAGE }}@${{ steps.build.outputs.digest }}

- name: Upload SBOM as artifact
  uses: actions/upload-artifact@... # pin to SHA
  with:
    name: sbom
    path: sbom.cdx.json
```

### For npm packages

```yaml
# npm provenance is built-in - just add --provenance flag
- name: Publish with provenance
  run: npm publish --provenance
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```
This requires `id-token: write` permission and publishes to npm with a Sigstore provenance statement.

### For source code

```yaml
- name: Generate source SBOM
  run: |
    syft dir:. -o cyclonedx-json > sbom-source.cdx.json

- name: Upload SBOM
  uses: actions/upload-artifact@... # pin to SHA
  with:
    name: sbom-source
    path: sbom-source.cdx.json
```

Make the actual workflow file changes for the user.

## Step 4: Add SLSA provenance (optional but recommended)

Walk through adding SLSA provenance using the official SLSA GitHub Generator:

```yaml
# This is a separate, isolated workflow - it builds and signs in a hardened builder
provenance:
  needs: [build]
  permissions:
    actions: read
    id-token: write
    contents: write
  uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
  with:
    base64-subjects: ${{ needs.build.outputs.digest }}
```

Explain that SLSA L3 provenance means the build ran on a hosted, hardened builder with non-falsifiable provenance - the strongest guarantee available.

## Step 5: Verify SBOMs and provenance

Show how to verify:
```bash
# Verify SBOM was attached
cosign verify-attestation --type cyclonedx $REGISTRY/$IMAGE@sha256:...

# Verify SLSA provenance
slsa-verifier verify-image $REGISTRY/$IMAGE@sha256:... \
  --source-uri github.com/OWNER/REPO
```

## Step 6: Ongoing SBOM management

Explain:
- SBOMs should be generated on every release, not just once
- Consider uploading SBOMs to a dependency tracking platform (Dependency-Track, Grype, etc.)
- Set up alerts for new CVEs against your SBOM inventory
- SBOMs can be required by admission controllers before deploying
