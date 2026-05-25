---
inclusion: manual
---

# Stage 11: Infrastructure as Code (Terraform)

## Goal
Define your entire EKS infrastructure as code — reproducible, version-controlled, reviewable. Move from eksctl (quick and dirty) to Terraform (production-grade).

## Why Terraform After eksctl?

| | eksctl | Terraform |
|--|--------|-----------|
| Speed to first cluster | Fast (one command) | Slower (more config) |
| Customization | Limited | Full control |
| State management | None (imperative) | State file (declarative) |
| Multi-resource | EKS only | VPC + EKS + IAM + everything |
| Team collaboration | Hard | State locking, plan/apply workflow |
| Production use | Prototyping | Standard |

## Terraform Basics

### Core Concepts
- **Provider**: plugin that talks to a cloud API (aws, kubernetes, helm)
- **Resource**: a thing to create (aws_eks_cluster, aws_vpc, etc.)
- **Data source**: read existing infrastructure
- **Module**: reusable group of resources
- **State**: Terraform's record of what it created (stored in S3)
- **Plan**: preview of changes before applying
- **Apply**: make the changes

### Project Structure
```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── addons/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── backend.tf
```

### Backend (Remote State)
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "eks/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # State locking
    encrypt        = true
  }
}
```

## VPC Module

```hcl
# modules/vpc/main.tf
variable "cluster_name" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "azs" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

locals {
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = var.vpc_cidr

  azs             = var.azs
  private_subnets = local.private_subnets
  public_subnets  = local.public_subnets

  enable_nat_gateway   = true
  single_nat_gateway   = false  # One per AZ for HA (true for cost savings in dev)
  enable_dns_hostnames = true
  enable_dns_support   = true

  # Tags required for EKS
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1  # For public load balancers
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1  # For internal load balancers
    "karpenter.sh/discovery"          = var.cluster_name
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

output "vpc_id" {
  value = module.vpc.vpc_id
}

output "private_subnet_ids" {
  value = module.vpc.private_subnets
}

output "public_subnet_ids" {
  value = module.vpc.public_subnets
}
```

## EKS Module

```hcl
# modules/eks/main.tf
variable "cluster_name" {
  type = string
}

variable "cluster_version" {
  type    = string
  default = "1.35"
}

variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = var.vpc_id
  subnet_ids = var.subnet_ids

  # Control plane access
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # EKS Add-ons
  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
      configuration_values = jsonencode({
        enableNetworkPolicy = "true"
      })
    }
    aws-ebs-csi-driver = {
      most_recent = true
    }
    eks-pod-identity-agent = {
      most_recent = true
    }
  }

  # Managed Node Groups
  eks_managed_node_groups = {
    general = {
      instance_types = ["t3.medium", "t3.large"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3

      labels = {
        role = "general"
      }

      tags = {
        "karpenter.sh/discovery" = var.cluster_name
      }
    }
  }

  # Access entries (IAM → K8s RBAC)
  access_entries = {
    admin = {
      principal_arn = "arn:aws:iam::123456789:role/AdminRole"
      policy_associations = {
        admin = {
          policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          access_scope = {
            type = "cluster"
          }
        }
      }
    }
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_certificate_authority_data" {
  value = module.eks.cluster_certificate_authority_data
}
```

## Environment Configuration

```hcl
# environments/production/main.tf
terraform {
  required_version = ">= 1.7"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

module "vpc" {
  source       = "../../modules/vpc"
  cluster_name = var.cluster_name
  vpc_cidr     = var.vpc_cidr
  azs          = var.azs
}

module "eks" {
  source          = "../../modules/eks"
  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
}
```

```hcl
# environments/production/terraform.tfvars
region          = "us-east-1"
cluster_name    = "production-eks"
cluster_version = "1.35"
vpc_cidr        = "10.0.0.0/16"
azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
```

## Terraform Workflow

```bash
# Initialize (download providers, configure backend)
terraform init

# Plan (preview changes)
terraform plan -out=plan.tfplan

# Apply (make changes)
terraform apply plan.tfplan

# Destroy (tear down everything)
terraform destroy
```

### Team Workflow
```
1. Developer creates branch
2. Makes Terraform changes
3. CI runs `terraform plan` on PR
4. Team reviews plan output
5. After merge, CI runs `terraform apply`
6. State is updated in S3
```

## Karpenter with Terraform

```hcl
# After EKS cluster is created, deploy Karpenter
module "karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  version = "~> 20.0"

  cluster_name = module.eks.cluster_name

  node_iam_role_additional_policies = {
    AmazonSSMManagedInstanceCore = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  }
}

resource "helm_release" "karpenter" {
  name       = "karpenter"
  namespace  = "karpenter"
  repository = "oci://public.ecr.aws/karpenter"
  chart      = "karpenter"
  version    = "1.1.0"

  create_namespace = true

  set {
    name  = "settings.clusterName"
    value = module.eks.cluster_name
  }

  set {
    name  = "settings.clusterEndpoint"
    value = module.eks.cluster_endpoint
  }
}
```

## State Management Best Practices

1. **Remote state in S3** with DynamoDB locking (prevents concurrent modifications)
2. **State per environment** (separate state files for dev/staging/prod)
3. **Never edit state manually** (use `terraform state` commands if needed)
4. **Encrypt state** (contains sensitive data like passwords)
5. **Lock state** (DynamoDB table prevents two people applying at once)

## Labs

### Lab 11.1: Terraform Basics
1. Create an S3 bucket + DynamoDB table for state
2. Write a simple Terraform config that creates a VPC
3. Run init, plan, apply
4. Modify the VPC, run plan — see the diff
5. Destroy everything

### Lab 11.2: EKS with Terraform
1. Create VPC module
2. Create EKS module using terraform-aws-modules/eks/aws
3. Deploy a full cluster with managed node groups
4. Configure kubectl to use the new cluster
5. Deploy a workload to verify

### Lab 11.3: Multi-Environment
1. Create dev and production environments using the same modules
2. Different instance sizes, replica counts, etc.
3. Apply dev first, then production
4. Verify both clusters are independent

### Lab 11.4: Add-ons with Terraform
1. Add Karpenter via Helm provider in Terraform
2. Add AWS Load Balancer Controller
3. Add External Secrets Operator
4. Apply and verify all add-ons are running

### Lab 11.5: Destroy and Recreate
1. Destroy the dev cluster completely
2. Recreate it from scratch with `terraform apply`
3. Verify everything comes back exactly as before
4. This proves your infrastructure is truly reproducible

## Self-Test Questions

1. Why use Terraform instead of eksctl for production? Name 3 advantages.
2. What's Terraform state? Where should you store it and why?
3. You and a colleague both run `terraform apply` at the same time. What could go wrong? How does DynamoDB locking prevent this?
4. What's the difference between `terraform plan` and `terraform apply`? Why should you always plan first?
5. You accidentally delete a resource from your Terraform code but it still exists in AWS. What happens on the next `terraform apply`?
6. What's a Terraform module? Why would you use one instead of putting everything in one file?
7. You run `terraform destroy` on your production cluster. What happens? How do you prevent accidental destruction?
8. What's the difference between `terraform import` and writing a resource from scratch? When would you use import?
9. In the VPC module, subnets are tagged with `kubernetes.io/role/elb`. Why? What breaks if you remove this tag?
10. You change the instance type in your EKS node group from t3.medium to t3.large. What does Terraform do? Is there downtime?
11. What's `terraform.tfvars`? How is it different from `variables.tf`?
12. You have dev and production environments using the same modules but different `tfvars`. How do you ensure a change to dev doesn't accidentally affect production?
13. What's the Helm provider in Terraform? Why would you deploy Helm charts via Terraform instead of `helm install`?
14. You run `terraform plan` and see "1 to destroy, 1 to create" for your EKS cluster. This would cause downtime. How do you investigate why?
15. What's `lifecycle { prevent_destroy = true }`? When would you use it?

## Checklist Before Moving On

- [ ] Understand Terraform core concepts (providers, resources, state, modules)
- [ ] Can set up remote state with S3 + DynamoDB
- [ ] Can create a VPC with public/private subnets using Terraform
- [ ] Can create an EKS cluster with managed node groups using Terraform
- [ ] Can manage EKS add-ons through Terraform
- [ ] Understand the plan/apply workflow
- [ ] Can structure Terraform for multiple environments
- [ ] Can deploy Helm charts via Terraform's Helm provider
- [ ] Know how to import existing resources into Terraform state
- [ ] Can destroy and recreate infrastructure reproducibly
