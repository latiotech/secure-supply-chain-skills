---
description: "Walkthrough: Set up commit and tag signing with GPG, SSH, or Sigstore gitsign"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, gpg:*, ssh-keygen:*, gitsign:*, which:*)
---

Interactive walkthrough to set up commit and tag signing so that every commit in your repository has a verified cryptographic identity. This prevents impersonation and ensures code provenance.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/credentials-configs.md` for reference configs.

## Why sign commits?

Git allows anyone to set `user.name` and `user.email` to any value. Without commit signing, there's no proof that a commit actually came from the person it claims to. An attacker with push access (or a compromised credential) can impersonate any contributor. Signed commits create a cryptographic link between the commit and a verified identity.

## Step 1: Assess current state

Check the project for:
- Whether any existing commits are signed: `git log --show-signature -5`
- Current git signing configuration: `git config --get commit.gpgsign`, `git config --get tag.gpgsign`, `git config --get gpg.format`
- Whether a signing key is already configured: `git config --get user.signingkey`
- Whether the repo has branch protection rules requiring signed commits (if it's a GitHub repo): `gh api repos/{owner}/{repo}/branches/main/protection/required_signatures` (may 404 if not set)
- Whether gitsign is installed: `which gitsign`

Present findings to the user.

## Step 2: Choose signing approach

Explain the three options:

### SSH signing (recommended for most teams)
- Uses existing SSH keys — no new tooling needed (Git 2.34+)
- GitHub, GitLab, and Bitbucket all support SSH signature verification
- Easiest setup: most developers already have SSH keys
- Keys can be added to GitHub's allowed signers for "Verified" badges

### GPG signing (traditional)
- Uses GPG keys — the original Git signing method
- Well-supported across all platforms and Git versions
- Requires GPG tooling installed and key management
- Better for teams that already use GPG for other purposes (encrypted email, etc.)

### Keyless signing with gitsign (zero key management)
- Uses OIDC identity (GitHub, Google, Microsoft) — no keys to manage at all
- Part of the Sigstore project — signatures logged in Rekor transparency log
- Best for open source projects or teams that want zero key management overhead
- Requires network access at signing time

Ask the user which approach fits their setup.

## Step 3: Configure signing

### SSH signing setup

1. **Check Git version** (requires 2.34+):
   ```bash
   git --version
   ```

2. **Use existing SSH key or generate one**:
   ```bash
   # List existing keys
   ls -la ~/.ssh/*.pub

   # Or generate a new one for signing
   ssh-keygen -t ed25519 -C "your_email@example.com"
   ```

3. **Configure Git to use SSH signing**:
   ```bash
   git config --global gpg.format ssh
   git config --global user.signingkey ~/.ssh/id_ed25519.pub
   git config --global commit.gpgsign true
   git config --global tag.gpgsign true
   ```

4. **Add the SSH key to GitHub for signature verification**:
   - Go to **Settings → SSH and GPG keys → New SSH key**
   - Set Key type to **Signing Key**
   - Paste the public key

5. **Set up allowed signers file** (for local verification):
   ```bash
   # Create allowed signers file
   echo "your_email@example.com $(cat ~/.ssh/id_ed25519.pub)" >> ~/.config/git/allowed_signers
   git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
   ```

### GPG signing setup

1. **Check for existing GPG keys**:
   ```bash
   gpg --list-secret-keys --keyid-format=long
   ```

2. **Generate a new GPG key** (if needed):
   ```bash
   gpg --full-generate-key
   # Choose: RSA and RSA, 4096 bits, key does not expire, your Git email
   ```

3. **Get the key ID**:
   ```bash
   gpg --list-secret-keys --keyid-format=long
   # Look for sec rsa4096/XXXXXXXXXXXXXXXX — the X's are your key ID
   ```

4. **Configure Git**:
   ```bash
   git config --global user.signingkey XXXXXXXXXXXXXXXX
   git config --global commit.gpgsign true
   git config --global tag.gpgsign true
   ```

5. **Add the GPG key to GitHub**:
   ```bash
   # Export the public key
   gpg --armor --export XXXXXXXXXXXXXXXX
   ```
   - Go to **Settings → SSH and GPG keys → New GPG key**
   - Paste the exported key

### Gitsign (keyless) setup

1. **Install gitsign**:
   ```bash
   # macOS
   brew install sigstore/tap/gitsign

   # Go install
   go install github.com/sigstore/gitsign@latest
   ```

2. **Configure Git to use gitsign**:
   ```bash
   # For the current repo only (recommended to start)
   git config gpg.x509.program gitsign
   git config gpg.format x509
   git config commit.gpgsign true
   git config tag.gpgsign true
   ```

   Or globally:
   ```bash
   git config --global gpg.x509.program gitsign
   git config --global gpg.format x509
   git config --global commit.gpgsign true
   git config --global tag.gpgsign true
   ```

3. **Configure your OIDC issuer** (optional — defaults to Sigstore public instance):
   ```bash
   # Use GitHub as OIDC provider
   git config --global gitsign.connectorID https://github.com/login/oauth
   ```

4. **Test it** — making a commit will open a browser for OIDC authentication.

## Step 4: Enforce signed commits on GitHub

Walk the user through requiring signed commits via branch protection:

### Using branch protection rules (classic)

1. Go to **Settings → Branches → Branch protection rules**
2. Select the main branch rule (or create one)
3. Enable **Require signed commits**

### Using repository rulesets (recommended — more flexible)

```bash
# Create a ruleset requiring signed commits on the default branch
gh api repos/{owner}/{repo}/rulesets -X POST --input - <<'EOF'
{
  "name": "Require signed commits",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main", "refs/heads/master"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "required_signatures" }
  ]
}
EOF
```

### For organizations

```bash
# Create an org-level ruleset (applies to all repos)
gh api orgs/{org}/rulesets -X POST --input - <<'EOF'
{
  "name": "Require signed commits",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main", "refs/heads/master"],
      "exclude": []
    },
    "repository_name": {
      "include": ["~ALL"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "required_signatures" }
  ]
}
EOF
```

Warn the user: enabling this will reject unsigned commits to protected branches. Make sure all contributors have signing configured first.

## Step 5: Verify the setup

After making changes:

1. **Make a test commit**:
   ```bash
   git commit --allow-empty -m "test: verify commit signing"
   ```

2. **Verify it's signed**:
   ```bash
   git log --show-signature -1
   ```

3. **Push and check GitHub** — the commit should show a "Verified" badge.

4. **For gitsign**, verify against the Rekor transparency log:
   ```bash
   gitsign verify --certificate-identity=your_email@example.com \
     --certificate-oidc-issuer=https://github.com/login/oauth HEAD
   ```

## Next steps

- Run `/setup-tag-rulesets` to also protect release tags from tampering
- Run `/harden-credentials` to ensure no leaked credentials could be used to push unsigned commits
- Run `/setup-image-signing` to extend signing to container images
