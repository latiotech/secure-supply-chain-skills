---
description: Pin base images by digest, enforce non-root, and harden Dockerfiles
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(docker:*, hadolint:*, git:*, which:*, brew:*)
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

## Prerequisites

Dockerfiles were found. Check if `hadolint` is installed (`which hadolint`).

If **not installed**, tell the user:

> **Recommended: install hadolint** — a Dockerfile linter that catches security misconfigurations, unpinned base images, and bad practices that manual review misses.
>
> ```bash
> # macOS
> brew install hadolint
> # or via Docker
> docker pull hadolint/hadolint
> ```
>
> Once installed, re-run this command for scanner-driven results. Continuing with pattern-based analysis for now.

## Actions to Take

For each Dockerfile, work through these in order. **Make each change directly, explain what was done and why, then move to the next.**

### 1. Run Hadolint (CRITICAL — if installed)

If hadolint is installed, run it against each Dockerfile:

```bash
hadolint Dockerfile
```

Parse and report findings grouped by severity. Use Hadolint's output to prioritize the fixes below — address scanner findings first. If Hadolint flags an issue that a later step would also catch (e.g., unpinned base image, running as root), fix it here and note it as "confirmed by Hadolint" when you reach that step.

### 2. Pin base images by digest (CRITICAL)

Look for `FROM` directives. For each that uses a tag without a digest:

**Important: get the multi-arch manifest digest, not a platform-specific digest.** Using `docker pull` + `docker inspect` pins to your local platform's digest (e.g., `arm64` on a Mac), which will break CI builds on `linux/amd64`. Instead, use registry-level tools that return the manifest list digest:

1. **Preferred** — `docker buildx imagetools inspect` (no pull needed, returns the manifest list digest):
   ```bash
   docker buildx imagetools inspect {image}:{tag} --format '{{json .Manifest}}' | jq -r '.digest'
   ```

2. **Alternative** — `crane digest` (if [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane) is installed):
   ```bash
   crane digest {image}:{tag}
   ```

3. **Fallback** — if neither is available, add a `# TODO: pin to manifest digest` comment with the `docker buildx imagetools inspect` command. **Do NOT use `docker pull` + `docker inspect`** — this pins to a single platform and will break cross-platform builds.

- Replace the FROM line: `FROM node:18-alpine` → `FROM node:18-alpine@sha256:abc123...`
- Explain: tags can be overwritten to point at compromised images; digests are immutable. The manifest list digest works across all platforms.

For images using `:latest` or no tag at all, flag these prominently — they're the highest risk. Pin to a specific version tag AND its manifest digest.

### 3. Add non-root user (HIGH)

For each Dockerfile, check if there's a `USER` directive before `CMD`/`ENTRYPOINT`. If not:
- Add a non-root user creation block before the final CMD:
  ```dockerfile
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  USER appuser
  ```
- Adjust for the base image's package manager (adduser vs useradd)
- Explain: running as root inside containers means a container escape gives the attacker root on the host

### 4. Remove hardcoded secrets (CRITICAL)

Scan for hardcoded secrets, API keys, or credentials in:
- `ENV` directives with values that look like secrets
- `COPY` or `ADD` of `.env` files
- `ARG` with default secret values

For each finding:
- Remove the secret from the Dockerfile
- Add a comment explaining to use Docker secrets, build-time `--secret` mounts, or environment variables at runtime
- Check `.dockerignore` exists and includes `.env*`, `.git`, `node_modules`, etc.

If `.dockerignore` doesn't exist, create one with common patterns.

### 5. Suggest multi-stage builds (MEDIUM)

If the Dockerfile installs build tools (gcc, make, build-essential, -dev packages) in the final image:
- Convert to a multi-stage build with a builder stage and a minimal production stage
- Suggest distroless or Alpine as the production base
- Explain: build tools in production images increase attack surface dramatically

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
