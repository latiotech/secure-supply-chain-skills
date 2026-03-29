---
description: Remove unused dependencies and convert Dockerfiles to multi-stage builds to reduce attack surface
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm:*, pnpm:*, yarn:*, pip:*, go:*, cargo:*, bundle:*, docker:*, git:*, which:*, depcheck:*, dpdm:*)
argument-hint: [domain: packages|containers|all]
---

Reduce attack surface by removing unused dependencies and minimizing container images. **This command takes action by default** — it identifies dead dependencies and removes them, and converts single-stage Dockerfiles to multi-stage builds. Changes are explained as they are made.

If $1 is provided, limit to that domain. Otherwise, check both packages and containers.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/container-configs.md` for multi-stage build patterns.
Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/package-configs.md` for package manager configurations.

## Part 1: Unused Dependencies

### Detection

Find all dependency files:
- `package.json` → npm/pnpm/yarn
- `requirements.txt` / `pyproject.toml` → pip/uv
- `go.mod` → Go
- `Cargo.toml` → Rust
- `Gemfile` → Ruby

If no dependency files are found, skip to Part 2.

### Analysis

For each detected package manager, identify unused dependencies.

#### Node.js (npm/pnpm/yarn)

Check if `depcheck` is installed (`which depcheck`). If not, suggest it:

> **Recommended: install depcheck** — analyzes your project to find unused dependencies, missing dependencies, and unnecessary entries.
>
> ```bash
> npm install -g depcheck
> ```
>
> Continuing with import-based analysis for now.

If depcheck is available:
```bash
depcheck --json
```

Parse the output. For each unused dependency reported:
- Verify it's truly unused — check for dynamic imports, config file references, CLI tools, and build plugins that depcheck may miss
- Classify as `dependencies` vs `devDependencies`

If depcheck is not available, do import-based analysis:
- Read `package.json` to get the full dependency list
- For each dependency, search the codebase for `require('{pkg}')`, `from '{pkg}'`, `import '{pkg}'`, and common subpath patterns like `from '{pkg}/'`
- Also check config files that reference packages: `babel.config.*`, `jest.config.*`, `.eslintrc.*`, `webpack.config.*`, `vite.config.*`, `tsconfig.json` (for `compilerOptions.types`), `postcss.config.*`, `.prettierrc`
- Also check `package.json` scripts for CLI tool usage (e.g., `"build": "tsc"` means `typescript` is used)
- Mark any dependency with zero references as a candidate for removal

For each candidate:
- Flag packages that are commonly used implicitly (type definition packages like `@types/*`, babel presets/plugins, eslint plugins, jest transformers, postcss plugins) — these may not have direct imports. Check the relevant config file before flagging them as unused.
- Distinguish between `dependencies` and `devDependencies` — dev dependencies only need to be reachable from dev tooling, not application code

#### Python (pip/uv)

- Read `requirements.txt` or `pyproject.toml` to get the dependency list
- For each dependency, search `.py` files for `import {pkg}`, `from {pkg} import`, accounting for package names that differ from import names (e.g., `Pillow` is imported as `PIL`, `python-dateutil` as `dateutil`)
- Common mismatches to handle: `beautifulsoup4`→`bs4`, `scikit-learn`→`sklearn`, `python-dotenv`→`dotenv`, `Pillow`→`PIL`, `PyYAML`→`yaml`, `python-dateutil`→`dateutil`
- Also check `pyproject.toml` `[tool.*]` sections and config files for plugin references

#### Go

```bash
go mod tidy -v 2>&1
```

Go mod tidy already removes unused dependencies. Report what it removed.

#### Rust

- Read `Cargo.toml` `[dependencies]` and `[dev-dependencies]`
- For each dependency, search `.rs` files for `use {crate}::`, `extern crate {crate}`, and `{crate}::` usage patterns
- Account for crate names with hyphens being imported with underscores (`my-crate` → `my_crate`)

#### Ruby

```bash
bundle clean --dry-run
```

Also check for gems referenced only in `Gemfile` but never `require`d or used in the codebase.

### Actions

For each confirmed unused dependency:

1. **Show the dependency and explain why it appears unused** — list what was searched and that no references were found
2. **Remove it from the dependency file** — edit `package.json`, `requirements.txt`, `Cargo.toml`, etc.
3. **Do NOT remove dependencies that are ambiguous** — if the package is a plugin, type definition, or has known implicit usage, flag it for manual review instead of removing it

After removing unused dependencies:
- For Node.js: note that the user should run `npm install` / `pnpm install` / `yarn install` to update the lockfile
- For Python: note that the user should regenerate their lockfile if using pip-tools or uv
- For Go: `go mod tidy` already handled the lockfile
- For Rust: note that the user should run `cargo update`

## Part 2: Multi-Stage Docker Builds

### Detection

Find all Dockerfiles:
- `Dockerfile`
- `Dockerfile.*`
- `*.dockerfile`

If no Dockerfiles are found, skip this part.

### Analysis

For each Dockerfile, determine if it's a candidate for multi-stage conversion:

**Already multi-stage** — has multiple `FROM` directives with `AS` aliases. Skip these but check if the final stage is minimal (report if the final stage uses a full base image when distroless/alpine would work).

**Single-stage candidates** — look for indicators that the image contains build artifacts that don't belong in production:

1. **Build tools installed**: `apt-get install` / `apk add` of packages like `gcc`, `g++`, `make`, `build-essential`, `python3-dev`, `*-dev`, `git`, `curl`, `wget`
2. **Build commands run**: `npm run build`, `yarn build`, `pnpm build`, `go build`, `cargo build`, `pip wheel`, `mvn package`, `gradle build`, `tsc`, `webpack`, `vite build`
3. **Source code copied but only compiled output needed**: `COPY . .` followed by a build step, where the output goes to a `dist/`, `build/`, `out/`, `target/`, or `bin/` directory
4. **Package manager cache present**: `node_modules` with both dependencies and devDependencies, `.cargo/registry`, pip cache

### Actions

For each single-stage Dockerfile that's a candidate:

#### 1. Determine the application type and pick the right production base

| Application Type | Detect By | Production Base |
|-----------------|-----------|-----------------|
| Node.js | `CMD ["node"`, `package.json` copy | `node:{version}-alpine` or `gcr.io/distroless/nodejs{version}-debian12` |
| Go | `go build`, `go.mod` copy | `gcr.io/distroless/static-debian12` or `alpine` |
| Python | `pip install`, `requirements.txt` copy | `python:{version}-slim` |
| Rust | `cargo build`, `Cargo.toml` copy | `gcr.io/distroless/cc-debian12` or `alpine` |
| Java | `mvn`, `gradle`, `.jar` file | `gcr.io/distroless/java{version}-debian12` or `eclipse-temurin:{version}-jre-alpine` |
| Static site | `nginx.conf`, HTML output | `nginx:alpine` or `caddy:alpine` |

#### 2. Convert to multi-stage build

Create a builder stage and a production stage. The conversion must:

- **Preserve the build logic exactly** — copy all build-related steps into the builder stage unchanged
- **Copy only runtime artifacts to the production stage** — compiled output, production dependencies, config files needed at runtime
- **Preserve the existing base image tag/digest** in the builder stage — don't change what they're building with
- **Pin the production base image** using `docker buildx imagetools inspect` to get the manifest digest (follow the same approach as `/harden-containers`)
- **Keep the existing `USER`, `EXPOSE`, `ENV`, `CMD`/`ENTRYPOINT`** directives in the production stage
- **Add a non-root user** in the production stage if one isn't already configured

For Node.js specifically:
- Install all dependencies in the builder stage (`npm ci`)
- Run the build step
- In the production stage, copy only `dist/` (or the build output directory) and reinstall with `--omit=dev` or copy `node_modules` after pruning

For Go specifically:
- Build with `CGO_ENABLED=0` for a static binary
- The production stage can use `scratch` or `distroless/static` — no runtime needed

#### 3. Create or update .dockerignore

If `.dockerignore` doesn't exist, create one. Ensure it includes:
```
.git
node_modules
dist
build
target
*.md
.env*
.github
```

Adjust based on what's actually in the project. Don't add entries that would break the build.

#### 4. Flag but do NOT auto-convert complex cases

Do not auto-convert Dockerfiles that:
- Use `docker-compose` volume mounts that depend on the full source tree being present
- Have conditional logic (`ARG`-driven `RUN` branches) that makes the build non-linear
- Are explicitly development Dockerfiles (`Dockerfile.dev`, `docker-compose.override.yml` targets)
- Run multiple processes (supervisord, multiple CMD scripts)

Instead, flag these with an explanation of what would need to change and why.

## Output

```
## Minimize Summary

### Unused Dependencies Removed
- [x] Removed `lodash` from dependencies (no imports found in src/)
- [x] Removed `@types/express` from devDependencies (express not used)
- [x] Flagged `babel-plugin-transform-runtime` for manual review (babel config plugin)

### Dockerfiles Converted to Multi-Stage
- [x] Dockerfile — split into builder (node:18-alpine) + production (node:18-alpine@sha256:...)
  - Build tools no longer in production image: gcc, make, python3
  - Estimated image size reduction: significant (full node + build tools → alpine runtime only)
- [x] Dockerfile.api — split into builder (golang:1.22) + production (distroless/static)

### Not Converted (Manual Review)
- [ ] Dockerfile.dev — development Dockerfile, skipped
- [ ] Dockerfile.worker — uses supervisord for multiple processes, needs manual refactor

### Manual Steps Needed
- [ ] Run `npm install` to update lockfile after dependency removal
- [ ] Run `docker build` to verify multi-stage builds work correctly
- [ ] Review flagged dependencies before removing

### Recommended Next Steps
- Run `/harden-containers` to pin base image digests and enforce non-root
- Run `/harden-packages` to pin remaining dependency versions
- Run `/update-pins` to check for newer versions of remaining dependencies
```
