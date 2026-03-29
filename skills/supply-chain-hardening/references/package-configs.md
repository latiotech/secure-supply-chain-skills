# Package Manager Security Configurations

Copy-paste configurations for hardening package managers across languages. Each section includes the config to add and what it does.

## npm / Node.js

### .npmrc - Secure defaults

```ini
# Disable pre/post install scripts (blocks malware like CanisterWorm)
ignore-scripts=true

# Require exact versions when installing
save-exact=true

# Require package-lock.json to be in sync
package-lock=true

# Audit on every install
audit=true

# Set a registry scope to prevent dependency confusion
# @yourorg:registry=https://npm.pkg.github.com
```

After setting `ignore-scripts=true`, explicitly allow scripts only for packages that need them:

```json
// package.json
{
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  },
  "allowScripts": {
    "esbuild": true,
    "sharp": true
  }
}
```

### package.json - Pin exact versions

```json
{
  "dependencies": {
    "express": "4.21.2",
    "lodash": "4.17.21"
  }
}
```

Never use `^` or `~` in production. Use `npm install --save-exact` or set `save-exact=true` in `.npmrc`.

## pnpm

### .npmrc - pnpm secure settings

```ini
# Disable all lifecycle scripts
ignore-scripts=true

# Pin exact versions
save-exact=true

# Require lockfile to be up to date
frozen-lockfile=true

# Only allow packages from specific registries per scope
# @yourorg:registry=https://npm.pkg.github.com

# Verify package integrity
strict-peer-dependencies=true
```

### pnpm-workspace.yaml - Allowed scripts

```yaml
# pnpm v9+ supports per-package script allowlists
onlyBuiltDependencies:
  - esbuild
  - sharp
  - bcrypt
```

### CI pipeline usage

```bash
# Always use frozen lockfile in CI
pnpm install --frozen-lockfile

# Audit before build
pnpm audit --audit-level=high
```

## Yarn (v4 / Berry)

### .yarnrc.yml - Secure defaults

```yaml
# Pin exact versions
defaultSemverRangePrefix: ""

# Enable strict mode
enableStrictSettings: true

# Disable lifecycle scripts globally
enableScripts: false

# Checksum verification
checksumBehavior: throw
```

## Python / pip

### pip.conf - Secure defaults

```ini
[global]
# Require hashes for all packages
require-hashes = true

# Only install from a known index
index-url = https://pypi.org/simple
no-cache-dir = true

[install]
# Never run setup.py install (use wheels only)
no-build-isolation = false
```

### requirements.txt - Pinned with hashes

```txt
# Generate with: pip-compile --generate-hashes requirements.in
flask==3.1.0 \
    --hash=sha256:5f1c7... \
    --hash=sha256:8a3b2...
requests==2.32.3 \
    --hash=sha256:70761...
```

### pip-tools workflow

```bash
# Install pip-tools
pip install pip-tools

# Compile pinned requirements with hashes from requirements.in
pip-compile --generate-hashes --output-file=requirements.txt requirements.in

# Install from pinned file
pip install --require-hashes -r requirements.txt
```

### uv - Modern Python package manager

```bash
# uv automatically creates lockfiles with hashes
uv lock

# Install from lockfile (equivalent to frozen-lockfile)
uv sync --frozen

# Audit dependencies
uv pip audit
```

## Go

### go.sum - Already pins by default

Go modules pin dependencies via `go.sum` with cryptographic hashes. Key settings:

```bash
# Enable module verification via Go checksum database
export GONOSUMCHECK=""  # empty = verify everything
export GOFLAGS="-mod=vendor"  # vendor dependencies for hermetic builds

# Use a private proxy for internal modules
export GOPROXY="https://proxy.yourorg.com,https://proxy.golang.org,direct"
export GONOSUMDB="yourorg.com/*"
export GOPRIVATE="yourorg.com/*"
```

### Verify and vendor

```bash
# Verify checksums
go mod verify

# Vendor all dependencies for reproducible builds
go mod vendor

# Tidy and verify
go mod tidy && go mod verify
```

## Ruby / Bundler

### .bundle/config

```yaml
BUNDLE_FROZEN: "1"           # Require Gemfile.lock to be up to date
BUNDLE_DISABLE_EXEC_LOAD: "1"  # Disable exec loading
```

### Gemfile - Pin exact versions

```ruby
gem 'rails', '= 7.2.2'
gem 'puma', '= 6.5.0'
```

## Rust / Cargo

Cargo.lock already pins exact versions. Ensure it's committed:

```bash
# In .gitignore, do NOT ignore Cargo.lock for applications
# (libraries may choose to ignore it)
```

### cargo-audit

```bash
# Install
cargo install cargo-audit

# Audit dependencies for known vulnerabilities
cargo audit

# In CI
cargo audit --deny warnings
```

## Java / Maven

### pom.xml - Pin versions and verify checksums

```xml
<!-- Enforce exact versions -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-enforcer-plugin</artifactId>
  <version>3.5.0</version>
  <executions>
    <execution>
      <id>enforce-versions</id>
      <goals><goal>enforce</goal></goals>
      <configuration>
        <rules>
          <requirePluginVersions/>
          <bannedDependencies>
            <excludes>
              <exclude>*:*:*-SNAPSHOT</exclude>
            </excludes>
          </bannedDependencies>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

### Gradle - Dependency verification

```bash
# Generate verification metadata
gradle --write-verification-metadata sha256

# This creates gradle/verification-metadata.xml with checksums
# Gradle will verify checksums on every build
```

## General CI/CD Patterns

### Pre-merge checks

```yaml
# GitHub Actions example
- name: Check for new dependencies
  run: |
    # Fail if lockfile changed without package.json change
    if git diff --name-only origin/main | grep -q "package-lock.json"; then
      if ! git diff --name-only origin/main | grep -q "package.json"; then
        echo "::error::Lockfile changed without package.json change - possible supply chain risk"
        exit 1
      fi
    fi
```

### Package cooldown enforcement

```yaml
# Block packages published in the last 7 days
- name: Check package age
  run: |
    npx is-old-enough --days 7 || exit 1
```
