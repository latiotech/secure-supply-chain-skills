---
description: "Walkthrough: Set up Kubernetes admission control to enforce image policies"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(kubectl:*, helm:*, kustomize:*, git:*)
---

Interactive walkthrough to set up Kubernetes admission controllers (Kyverno or OPA Gatekeeper) that enforce container image policies - requiring signed images, blocking unscanned images, and preventing root containers.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/container-configs.md` for admission control reference configs.

## Why admission control?

Without admission control, anyone who can deploy to your cluster can run any image - including compromised ones. Admission controllers act as a policy gate: if the image isn't signed, scanned, or from an approved registry, it doesn't run.

## Step 1: Assess current state

Check the project for:
- Kubernetes manifests (Deployments, StatefulSets, DaemonSets, CronJobs)
- Helm charts or Kustomize overlays
- Existing admission webhooks or policy engines
- Whether images are referenced by digest or tag
- Whether image signing is set up (if not, suggest `/setup-image-signing` first)

Ask the user:
- Which cluster(s) do you want to protect?
- Are you already using Kyverno, OPA Gatekeeper, or Kubewarden?
- Do you have image signing set up? (Required for signature verification policies)

## Step 2: Choose a policy engine

Explain the options:

### Kyverno (recommended for most teams)
- Kubernetes-native, YAML-based policies (no Rego)
- Built-in image verification support (Cosign, Notation)
- Lower learning curve
- Install: `helm install kyverno kyverno/kyverno -n kyverno --create-namespace`

### OPA Gatekeeper
- General-purpose policy engine using Rego language
- More powerful but steeper learning curve
- Better if you already use OPA elsewhere
- Install: `helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system --create-namespace`

### Kubewarden
- WebAssembly-based - write policies in any Wasm-compatible language
- Good for teams that prefer real programming languages over YAML/Rego

Ask which they prefer and proceed accordingly.

## Step 3: Install the policy engine

Walk through installation for the chosen engine. Provide the exact Helm commands and verify the installation is healthy.

## Step 4: Create policies

Walk the user through creating policies for their chosen engine. For each policy, explain what it enforces and create the YAML manifest.

### Policy 1: Require signed images (Kyverno example)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce  # or Audit to start
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences: ["your-registry.io/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/OWNER/REPO/.github/workflows/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

Ask for their registry and repo details to fill in the template.

### Policy 2: Block root containers

### Policy 3: Restrict image registries

### Policy 4: Require resource limits

For each policy:
- Start in **Audit** mode so existing workloads aren't disrupted
- Show how to check violations: `kubectl get policyreport -A`
- Switch to **Enforce** after verifying no false positives

## Step 5: Test policies

Walk through testing:
1. Try deploying a pod with an unsigned image - should be blocked (or reported in Audit mode)
2. Try deploying a pod running as root - should be blocked
3. Deploy a properly signed image - should succeed

## Step 6: Roll out

Explain the recommended rollout:
1. Deploy policies in **Audit** mode to all namespaces
2. Review policy reports for 1-2 weeks
3. Fix any violations in existing workloads
4. Switch to **Enforce** mode
5. Add policies to your GitOps repo so they're version-controlled

## Next steps

- Run `/setup-image-signing` if not already done
- Run `/setup-sbom` to attach SBOMs to images and verify them at admission
