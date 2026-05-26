---
inclusion: auto
description: AWS CLI profile configuration — enforces --profile eks-learning on all AWS commands
---

# AWS CLI Profile: eks-learning

## ⚠️ MANDATORY RULE (applies to ALL stages, ALL commands)

**Every AWS CLI command generated or suggested in this project MUST include `--profile eks-learning`.**

This is non-negotiable. No exceptions. Applies to:
- `aws` commands (any subcommand: eks, ec2, iam, s3, sts, cloudformation, etc.)
- `eksctl` commands (use `--profile eks-learning`)
- Any script, lab, or example that calls AWS APIs
- Terraform AWS provider blocks (use `profile = "eks-learning"`)
- Environment variable alternative: `export AWS_PROFILE=eks-learning`

If you forget the profile flag, the command is WRONG. Always include it.

## Profile Configuration

| Setting | Value |
|---------|-------|
| Profile name | `eks-learning` |
| Default region | `ap-south-1` (Mumbai) |
| Output format | `json` |
| AWS Account | `851060550361` |
| IAM User | `nextcloud-poc` |

## Setup Instructions

If the profile does not exist, create it:

```bash
aws configure --profile eks-learning
```

When prompted:
- **AWS Access Key ID**: (your key)
- **AWS Secret Access Key**: (your secret)
- **Default region name**: `ap-south-1`
- **Default output format**: `json`

## Verification

```bash
aws sts get-caller-identity --profile eks-learning
```

## Examples (correct usage)

```bash
# AWS CLI
aws eks list-clusters --profile eks-learning
aws ec2 describe-vpcs --profile eks-learning
aws iam list-roles --profile eks-learning

# eksctl
eksctl create cluster --profile eks-learning ...
eksctl get clusters --profile eks-learning

# Terraform provider block
provider "aws" {
  region  = "ap-south-1"
  profile = "eks-learning"
}

# Session export (alternative)
export AWS_PROFILE=eks-learning
```

## Kiro Enforcement Rule

When generating or suggesting ANY command that touches AWS in this project:
1. ALWAYS append `--profile eks-learning` (or set the profile in provider/env)
2. ALWAYS use region `ap-south-1` unless explicitly overridden
3. If the profile is not configured, tell the user to run `aws configure --profile eks-learning` before proceeding
4. This rule applies across ALL stages (0-13), ALL labs, ALL examples
