---
description: "Walkthrough: Add runtime detection and monitoring to CI runners"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*)
---

Interactive walkthrough to add runtime detection and monitoring to GitHub Actions runners. This catches malicious behavior at runtime - credential theft, memory dumps, data exfiltration - even when the malicious code made it past static checks.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/actions-configs.md` for runtime detection reference configs.

## Why runtime monitoring?

Static analysis catches known bad patterns. But sophisticated supply chain attacks (like TeamPCP) use novel techniques:
- Dumping runner process memory via `/proc/[pid]/mem` to steal in-memory credentials
- Sweeping 50+ credential file paths (`.aws/credentials`, `.ssh/`, `.npmrc`, etc.)
- Exfiltrating stolen data to attacker-controlled endpoints

Runtime monitoring detects this behavior as it happens, regardless of how the malicious code got in.

## Step 1: Assess current state

Check the project for:
- Existing GitHub Actions workflows
- Whether StepSecurity Harden Runner is already in use
- Whether workflows have explicit network egress requirements
- What secrets are available to workflows
- Whether self-hosted or GitHub-hosted runners are used

Present findings and ask:
- Do you use GitHub-hosted or self-hosted runners?
- Do you know which external endpoints your workflows need to reach?

## Step 2: Add StepSecurity Harden Runner

Walk the user through adding Harden Runner to their workflows. This is the fastest path to runtime visibility.

### Start with Audit mode

For each workflow, add Harden Runner as the **first step** in each job:

```yaml
steps:
  - name: Harden Runner
    uses: step-security/harden-runner@17d0e2bd7d51742c71671bd19fa12bdc9d40a3d6 # v2.8.1
    with:
      egress-policy: audit
```

Make the actual workflow file changes.

Explain:
- In `audit` mode, Harden Runner logs all network egress and file access but doesn't block anything
- This lets you build a baseline of expected behavior before blocking unexpected activity
- Results are visible in the StepSecurity dashboard and as workflow annotations

### After 1-2 weeks, switch to Block mode

Explain how to transition:
1. Review the audit logs to see which endpoints each workflow contacts
2. Build an allowlist of expected endpoints
3. Switch to block mode with the allowlist:

```yaml
- name: Harden Runner
  uses: step-security/harden-runner@17d0e2bd7d51742c71671bd19fa12bdc9d40a3d6 # v2.8.1
  with:
    egress-policy: block
    allowed-endpoints: >
      github.com:443
      api.github.com:443
      registry.npmjs.org:443
      ghcr.io:443
```

Note: The specific allowed endpoints depend on the workflow. The audit mode logs will show exactly what's needed.

## Step 3: Self-hosted runner hardening (if applicable)

If the user uses self-hosted runners, walk through additional protections:

### Ephemeral runners
- Each job should get a fresh runner VM that's destroyed after use
- This prevents credential persistence and cross-job contamination
- For GitHub-hosted runners, this is automatic
- For self-hosted: use `--ephemeral` flag with the runner agent, or use solutions like:
  - **Actions Runner Controller (ARC)** with Kubernetes - spins up pods per job
  - **Philips self-hosted runner images** - pre-hardened AMIs
  - **Runs-on** - autoscaling self-hosted runners on AWS

### Falco for deeper detection

If the user runs self-hosted runners and wants deeper detection:

```yaml
# Falco rules for CI runner monitoring
# Deploy Falco on the runner hosts

# Detect /proc/mem dumps (TeamPCP technique)
- rule: Dump Memory using /proc/ Filesystem
  desc: Detect reading from /proc/[pid]/mem
  condition: >
    open_read and fd.name startswith /proc/ and fd.name contains /mem
  output: "Memory dump detected (user=%user.name command=%proc.cmdline file=%fd.name)"
  priority: CRITICAL

# Detect credential file sweeps
- rule: Read Sensitive Credential Files
  desc: Detect reading SSH keys, cloud creds, .env files
  condition: >
    open_read and (fd.name startswith /home/ or fd.name startswith /root/)
    and (fd.name contains .ssh or fd.name contains .aws or fd.name contains .env)
  output: "Sensitive file read (user=%user.name file=%fd.name command=%proc.cmdline)"
  priority: WARNING
```

Explain that Falco is a CNCF runtime security tool that uses eBPF to detect suspicious kernel-level activity.

## Step 4: Network segmentation

Explain additional network-level protections:
- Use firewall rules to restrict runner egress to only required endpoints
- Block access to internal services from CI runners
- Use a forward proxy for all runner egress to log and filter traffic
- Consider running CI in an isolated VPC/network segment

## Step 5: Alerting

Walk through setting up alerts:
- StepSecurity provides built-in alerts for anomalous behavior
- Falco can send alerts to Slack, PagerDuty, or SIEM
- GitHub Actions workflow failure notifications for blocked egress

## Next steps

- Run `/harden-actions` to pin Actions and fix permissions (reduces the attack surface that runtime monitoring needs to catch)
- Run `/setup-oidc` to eliminate long-lived credentials from runners
- Consider deploying runtime detection on developer workstations as well (EDR)
