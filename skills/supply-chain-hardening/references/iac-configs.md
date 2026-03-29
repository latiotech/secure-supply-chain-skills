# Infrastructure-as-Code Hardening Configurations

Practical configs for securing Terraform, OpenTofu, and IaC modules.

## Pin Module and Provider Versions

```hcl
# BAD - floating version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
}

# GOOD - pinned to exact version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.16.0"
}

# BEST - pinned to exact version with integrity hash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.82.2"
    }
  }
}
```

### Lock file

```bash
# Generate/update .terraform.lock.hcl with hashes for all platforms
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64

# Commit .terraform.lock.hcl to version control
git add .terraform.lock.hcl
```

## Scan with Checkov

```bash
# Install
pip install checkov

# Scan a directory
checkov -d .

# Scan specific framework
checkov -d . --framework terraform

# Skip specific checks if justified
checkov -d . --skip-check CKV_AWS_18  # S3 access logging - may not apply

# Output SARIF for GitHub
checkov -d . -o sarif > checkov-results.sarif
```

### CI pipeline example

```yaml
- name: Run Checkov
  run: |
    pip install checkov
    checkov -d ./terraform --output-file-path console,sarif --soft-fail-on LOW
```

## Scan with tflint

```bash
# Install
brew install tflint  # or curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Initialize plugins
tflint --init

# Run
tflint --recursive
```

### .tflint.hcl

```hcl
plugin "aws" {
  enabled = true
  version = "0.33.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_required_providers" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}
```

## Lock Down Provisioners

```hcl
# AVOID local-exec and external provisioners
# These run arbitrary commands and are supply chain risk

# BAD
resource "null_resource" "setup" {
  provisioner "local-exec" {
    command = "curl -s https://example.com/script.sh | bash"
  }
}

# If you MUST use provisioners, restrict with Sentinel/OPA
```

### OPA policy to block provisioners

```rego
package terraform.provisioners

deny[msg] {
  resource := input.resource_changes[_]
  provisioner := resource.change.after.provisioner[_]
  provisioner.type == "local-exec"
  msg := sprintf("local-exec provisioner not allowed in %s", [resource.address])
}

deny[msg] {
  resource := input.resource_changes[_]
  provisioner := resource.change.after.provisioner[_]
  provisioner.type == "external"
  msg := sprintf("external provisioner not allowed in %s", [resource.address])
}
```

## State Security

```hcl
# Encrypted remote state with access controls
terraform {
  backend "s3" {
    bucket         = "yourorg-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/abc-123"
    dynamodb_table = "terraform-locks"
  }
}
```

Key state security checklist:
- Enable encryption at rest (S3 SSE-KMS, GCS CMEK, Azure Storage encryption)
- Enable versioning on the state bucket
- Restrict access via IAM - only CI/CD and admins
- Enable state locking (DynamoDB for AWS, built-in for GCS/Azure)
- Never commit state files to git
- Audit state access via CloudTrail / Cloud Audit Logs

## Drift Detection

```bash
# Manual drift check
terraform plan -detailed-exitcode
# Exit code 2 = drift detected

# In CI (scheduled)
- name: Drift Detection
  run: |
    terraform init
    terraform plan -detailed-exitcode
  # Alert on exit code 2
```

For automated drift detection at scale, use Spacelift, env0, or Firefly - they continuously reconcile and alert.

## Require Signed Commits for IaC Repos

```yaml
# Branch protection rule (or ruleset)
# Settings → Rules → Rulesets → Target: main branch
# Enable: "Require signed commits"
```

```bash
# Set up GPG signing locally
git config --global commit.gpgsign true
git config --global user.signingkey YOUR_KEY_ID
```
