---
description: "Walkthrough: Replace long-lived cloud credentials with OIDC in GitHub Actions"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*, gh:*, aws:*, gcloud:*, az:*)
---

Interactive walkthrough to replace long-lived cloud credentials (AWS keys, GCP service account keys, Azure secrets) with short-lived OIDC tokens in GitHub Actions. This eliminates the risk of credential theft from CI runners.

Read `${CLAUDE_PLUGIN_ROOT}/skills/supply-chain-hardening/references/actions-configs.md` for OIDC reference configs.

## Why OIDC?

Long-lived credentials stored as GitHub Secrets are a prime target. If a CI runner is compromised (via a malicious Action, dependency, or `pull_request_target` exploit), those credentials are stolen. OIDC tokens are short-lived, scoped to a single workflow run, and never stored as secrets.

## Step 1: Identify current credential usage

Search the project's workflow files for long-lived cloud credentials:
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`
- `GOOGLE_APPLICATION_CREDENTIALS` / `GCP_SA_KEY`
- `AZURE_CREDENTIALS` / `AZURE_CLIENT_SECRET`
- Any `${{ secrets.* }}` that reference cloud provider credentials

For each finding, show:
- The workflow file and line
- What the credentials are used for (deploy, publish, infra management, etc.)
- Whether the workflow has `id-token: write` permission already

Present a summary table of all credential usage before proceeding.

## Step 2: Choose cloud provider

Ask the user which cloud provider(s) they need to configure. Walk through the appropriate setup:

### AWS OIDC Setup

Walk the user through these steps:

1. **Create an OIDC identity provider in AWS IAM**
   ```
   Audience: sts.amazonaws.com
   Provider URL: https://token.actions.githubusercontent.com
   ```
   Provide the AWS CLI command:
   ```bash
   aws iam create-open-id-connect-provider \
     --url https://token.actions.githubusercontent.com \
     --client-id-list sts.amazonaws.com \
     --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
   ```

2. **Create an IAM role with a trust policy**
   Show the trust policy JSON scoped to their specific repo:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": {"Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"},
       "Action": "sts:AssumeRoleWithWebIdentity",
       "Condition": {
         "StringEquals": {
           "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
         },
         "StringLike": {
           "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:*"
         }
       }
     }]
   }
   ```
   Ask for their AWS account ID and repo name to fill in the template.

3. **Attach the minimum IAM permissions** the role needs based on what the workflow does.

4. **Update the workflow** - make the actual file changes:
   - Add `permissions: id-token: write, contents: read`
   - Replace credential-using steps with `aws-actions/configure-aws-credentials` using `role-to-assume`
   - Remove references to `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`

5. **Remind the user to delete the old secrets** from GitHub Settings → Secrets after verifying OIDC works.

### GCP OIDC Setup

Walk the user through:

1. **Create a Workload Identity Pool and Provider**
   ```bash
   gcloud iam workload-identity-pools create "github-pool" \
     --location="global" \
     --display-name="GitHub Actions Pool"

   gcloud iam workload-identity-pools providers create-oidc "github-provider" \
     --location="global" \
     --workload-identity-pool="github-pool" \
     --display-name="GitHub Provider" \
     --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
     --issuer-uri="https://token.actions.githubusercontent.com"
   ```

2. **Create a service account and bind it** to the workload identity pool, scoped to the repo.

3. **Update the workflow** - make the actual file changes:
   - Add `permissions: id-token: write, contents: read`
   - Use `google-github-actions/auth` with `workload_identity_provider` and `service_account`

4. **Remind the user to delete old GCP SA keys** from GitHub Secrets.

### Azure OIDC Setup

Walk the user through:

1. **Create a federated credential** on an Azure AD App Registration, scoped to the repo.
2. **Update the workflow** to use `azure/login` with OIDC.
3. **Remind the user to delete old Azure secrets.**

## Step 3: Test and verify

After making workflow changes:
- Suggest the user push the changes to a branch and run the workflow
- Explain how to check the Actions log for successful OIDC authentication
- Note that the old secrets should NOT be deleted until OIDC is confirmed working

## Step 4: Lock down further

After OIDC is working, suggest:
- Scope the OIDC trust policy to specific branches (e.g., `repo:OWNER/REPO:ref:refs/heads/main`)
- Scope to specific workflow files or environments for tighter control
- Remove all remaining long-lived cloud credentials from GitHub Secrets
