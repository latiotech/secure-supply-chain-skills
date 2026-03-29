# Container Image Hardening Configurations

Practical configs for securing Dockerfiles, image builds, and container registries.

## Dockerfile Hardening

### Pin base images by digest

```dockerfile
# BAD - tag can be moved to point at a compromised image
FROM node:18-alpine

# GOOD - pinned to specific digest
FROM node:18-alpine@sha256:a1b2c3d4e5f6...

# Find the current digest:
# docker pull node:18-alpine && docker inspect --format='{{index .RepoDigests 0}}' node:18-alpine
```

### Run as non-root

```dockerfile
FROM node:18-alpine@sha256:a1b2c3...

WORKDIR /app

# Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy and install deps as root
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --ignore-scripts

COPY . .

# Switch to non-root before running
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

### Multi-stage builds for minimal images

```dockerfile
# Build stage
FROM node:18-alpine@sha256:a1b2c3... AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --ignore-scripts
COPY . .
RUN pnpm build

# Production stage - minimal image
FROM gcr.io/distroless/nodejs18-debian12@sha256:d4e5f6...
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["dist/server.js"]
```

### Dockerfile linting with Hadolint

```bash
# Install
brew install hadolint  # or docker run --rm -i hadolint/hadolint < Dockerfile

# Run
hadolint Dockerfile

# In CI (GitHub Actions)
# - uses: hadolint/hadolint-action@v3.1.0  # Pin to SHA in practice
```

Key Hadolint rules to enforce:
- `DL3007` - Using latest tag (pin versions instead)
- `DL3006` - Missing tag on FROM (always specify version)
- `DL3002` - Last USER should not be root
- `DL3008` - Pin versions in apt-get install
- `DL3018` - Pin versions in apk add

## Image Scanning

### Grype - scan locally

```bash
# Install
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# Scan an image
grype myapp:latest

# Fail on high/critical vulnerabilities
grype myapp:latest --fail-on high

# Scan and output SARIF for GitHub
grype myapp:latest -o sarif > grype-results.sarif
```

### Trivy - scan in CI

```bash
# Scan image
trivy image myapp:latest

# Fail on critical
trivy image --severity CRITICAL --exit-code 1 myapp:latest

# Generate SBOM
trivy image --format cyclonedx --output sbom.json myapp:latest
```

### CI pipeline example

```yaml
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan with Grype
        run: |
          grype myapp:${{ github.sha }} --fail-on high

      - name: Lint Dockerfile
        run: hadolint Dockerfile
```

## Image Signing with Cosign

```bash
# Install cosign
brew install cosign  # or go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Keyless signing (uses OIDC - recommended)
cosign sign --yes ghcr.io/yourorg/myapp@sha256:abc123...

# Verify a signed image
cosign verify ghcr.io/yourorg/myapp@sha256:abc123... \
  --certificate-identity=your-ci@yourorg.iam.gserviceaccount.com \
  --certificate-oidc-issuer=https://accounts.google.com

# Attach SBOM to image
cosign attach sbom --sbom sbom.json ghcr.io/yourorg/myapp@sha256:abc123...
```

## Admission Control with Kyverno

### Require signed images

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/yourorg/*"
          attestors:
            - entries:
                - keyless:
                    subject: "your-ci@yourorg.iam.gserviceaccount.com"
                    issuer: "https://accounts.google.com"
```

### Block images running as root

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: non-root-containers
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Containers must not run as root"
        pattern:
          spec:
            containers:
              - securityContext:
                  runAsNonRoot: true
```

## Minimal Base Image Comparison

| Image | Size | Shell | Package Manager | libc | Best For |
|-------|------|-------|-----------------|------|----------|
| Alpine | ~5MB | Yes (ash) | apk | musl | General use, familiar workflow |
| Distroless | ~2-20MB | No | No | glibc | Production, no debug access needed |
| Wolfi (Chainguard) | ~5-15MB | Optional | apk | glibc | Best compatibility + security |
| Scratch | 0MB | No | No | None | Static Go/Rust binaries |
