---
description: Pin base images by digest, enforce non-root, and harden Dockerfiles
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(docker:*, hadolint:*, grype:*, trivy:*, cosign:*, git:*)
argument-hint: [Dockerfile-path]
---

Audit and harden container images for supply chain security. **This command takes action by default** - it pins base images to digests, adds non-root users, and fixes Dockerfile issues. Changes are explained as they are made.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/container-configs.md` for detailed configurations.

## Detection

Find all Dockerfiles in the project:
- `Dockerfile`
- `Dockerfile.*` (e.g., `Dockerfile.prod`, `Dockerfile.dev`)
- `*.dockerfile`
- `docker-compose.yml` / `docker-compose.*.yml` (for image references)

If $1 is provided, focus on that specific Dockerfile.

## Actions to Take

For each Dockerfile, work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Pin base images by digest (CRITICAL)

Look for `FROM` directives. For each that uses a tag without a digest:
- Run `docker pull {image}:{tag}` and then `docker inspect --format='{{index .RepoDigests 0}}' {image}:{tag}` to get the current digest
- If docker is not available, note the image and tag and add a `# TODO: pin to digest` comment with the command to run
- Replace the FROM line: `FROM node:18-alpine` → `FROM node:18-alpine@sha256:abc123...`
- Explain: tags can be overwritten to point at compromised images; digests are immutable

For images using `:latest` or no tag at all, flag these prominently - they're the highest risk. Pin to a specific version tag AND its digest.

### 2. Add non-root user (HIGH)

For each Dockerfile, check if there's a `USER` directive before `CMD`/`ENTRYPOINT`. If not:
- Add a non-root user creation block before the final CMD:
  ```dockerfile
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  USER appuser
  ```
- Adjust for the base image's package manager (adduser vs useradd)
- Explain: running as root inside containers means a container escape gives the attacker root on the host

### 3. Remove hardcoded secrets (CRITICAL)

Scan for hardcoded secrets, API keys, or credentials in:
- `ENV` directives with values that look like secrets
- `COPY` or `ADD` of `.env` files
- `ARG` with default secret values

For each finding:
- Remove the secret from the Dockerfile
- Add a comment explaining to use Docker secrets, build-time `--secret` mounts, or environment variables at runtime
- Check `.dockerignore` exists and includes `.env*`, `.git`, `node_modules`, etc.

If `.dockerignore` doesn't exist, create one with common patterns.

### 4. Suggest multi-stage builds (MEDIUM)

If the Dockerfile installs build tools (gcc, make, build-essential, -dev packages) in the final image:
- Convert to a multi-stage build with a builder stage and a minimal production stage
- Suggest distroless or Alpine as the production base
- Explain: build tools in production images increase attack surface dramatically

### 5. Run available scanners

If any of these tools are installed, run them:
- `hadolint {Dockerfile}` - lint for best practices
- `grype {image} --fail-on high` - scan for vulnerabilities (if image is built)
- `trivy image {image}` - comprehensive scan (if image is built)

Report results grouped by severity.

## Output

After making all changes, provide a summary:

```
## Container Hardening Summary

### Changes Made
- [x] Pinned N base images to digests in [files]
- [x] Added non-root user to N Dockerfiles
- [x] Created .dockerignore
- [x] Converted [file] to multi-stage build

### Manual Steps Needed
- [ ] Pin digest for [image] (docker not available - run the provided command)
- [ ] Remove secret from [file:line] and use Docker secrets instead

### Recommended Next Steps
- Run `/setup-image-signing` to sign images with Cosign/Sigstore
- Run `/setup-admission-control` to enforce signed-only images in your cluster
```
