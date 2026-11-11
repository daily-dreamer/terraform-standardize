---
name: terraform-standardize
description: >-
  Standardize raw, brownfield, or Terraformer-imported Terraform code (AWS, GCP, Azure)
  into production-ready HashiCorp standards: isolates provider.tf, versions.tf, and backend.tf,
  parameterizes hardcoded strings into typed variables.tf and locals.tf, replaces static cloud IDs
  with dynamic references, enforces ForceNew and zero-drift guardrails with moved blocks, and runs
  canonical HCL formatting, validation, and plan diff summaries.
license: Apache-2.0
metadata:
  version: v1.0.0
---

# Terraform Standardization Skill

This skill guides the agent in refactoring raw, brownfield, imported (e.g. from Terraformer or `terraform import`), or legacy Terraform code into modular, production-ready HashiCorp standard architecture.

## When to Use This Skill

Activate this skill when:
- The user asks to "standardize", "refactor", "clean up", or "modularize" Terraform configurations.
- Raw Terraform code contains hardcoded strings, magic numbers, or static cloud IDs (`ami-*`, `vpc-*`, `subnet-*`, `sg-*`, `arn:*`).
- Monolithic single-file configurations (`main.tf` with everything) need separation into standard multi-file layouts (`provider.tf`, `variables.tf`, `locals.tf`, `outputs.tf`, `backend.tf`).
- Cloud resources need standardized tagging taxonomies (e.g. `Environment`, `Project`, `ManagedBy = "terraform"`).
- The codebase lacks variable definitions, type constraints, or description blocks.

---

## Standardization Workflow

Follow this 5-phase procedure when standardizing any Terraform codebase:

```
┌────────────────────────────────────────────────────────┐
│ Phase 1: Audit & Identify Code Smells                  │
│ • Detect hardcoded literals, IDs, regions, and CIDRs   │
│ • Inspect existing file structure vs standard layout   │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│ Phase 2: Establish Standard File Architecture          │
│ • Extract provider.tf, versions.tf, and backend.tf     │
│ • Create variables.tf, locals.tf, and outputs.tf       │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│ Phase 3: Parameterize Literals into Variables & Locals │
│ • Extract environment/region variables with types      │
│ • Build local.common_tags for unified resource tagging │
│ • Create terraform.tfvars.example                      │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│ Phase 4: Dynamic Reference Linking & Renaming          │
│ • Replace static resource IDs with dynamic references  │
│ • Replace hardcoded AMIs/subnets with data sources     │
│ • Standardize resource labels (snake_case, no types)   │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│ Phase 5: Format, Validate, and Verify                  │
│ • Run `terraform fmt -recursive`                       │
│ • Run `terraform validate`                             │
│ • Verify with `terraform plan` (zero drift)            │
└────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Instructions

### 1. Phase 1: Audit & Discovery

Before making edits, scan the target directory to identify:
1. **CLI Availability**: Run `terraform version` (or `tofu version` for OpenTofu) to determine whether CLI formatting, validation, and plan checks can run in Phase 5.
2. **Missing standard files**: Check if `versions.tf`, `provider.tf`, `variables.tf`, `locals.tf`, and `outputs.tf` exist.
3. **Hardcoded IDs & literals**:
   - Cloud identifiers: `ami-[a-f0-9]+`, `vpc-[a-f0-9]+`, `subnet-[a-f0-9]+`, `sg-[a-f0-9]+`, `igw-[a-f0-9]+`, full GCP self-links (`https://www.googleapis.com/compute/...`), static `project` IDs
   - CIDR blocks: `"10.0.0.0/16"`, `"0.0.0.0/0"`
   - Instance types: `"t3.micro"`, `"m5.large"`, `"e2-standard-4"`
   - Static regions & account numbers: `"us-east-1"`, `"us-central1"`, `"123456789012"`
   - Raw credentials or unencrypted secrets
4. **Formatting Baseline**: If Terraform is installed, check if files comply with `terraform fmt -check -recursive`.

### 2. Phase 2: Create Standard Core Files

Ensure the target configuration is split into distinct, single-responsibility files:

#### `provider.tf` / `versions.tf`
Centralize Terraform settings, minimum CLI version, and required providers (adapt for AWS, GCP, or Azure):
```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    # GCP example:
    # google = {
    #   source  = "hashicorp/google"
    #   version = "~> 5.5"
    # }
  }
}

# provider.tf (AWS)
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = local.common_tags
  }
}

# provider.tf (GCP alternative)
# provider "google" {
#   project        = var.project_id
#   region         = var.region
#   default_labels = local.common_labels
# }
```

> **Single Tagging/Labeling Strategy**: Use **either** provider-level `default_tags` / `default_labels` (putting only resource-specific keys on individual resources) **or** `merge(local.common_tags, ...)` / `merge(local.common_labels, ...)` on each resource — never duplicate both.

#### `backend.tf` (if remote state is used)
Separate state storage configuration (`s3` for AWS, `gcs` for GCP, `azurerm` for Azure):
```hcl
terraform {
  backend "s3" {
    bucket         = "myorg-terraform-state"
    key            = "services/app/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# GCP example:
# terraform {
#   backend "gcs" {
#     bucket = "myorg-terraform-state"
#     prefix = "services/app"
#   }
# }
```

### 3. Phase 3: Convert Plain Strings to Variables and Locals

Extract all hardcoded parameters into `variables.tf` and `locals.tf`:

#### `variables.tf` Rules:
- **Always declare `type`**: (`string`, `number`, `bool`, `list(string)`, `map(string)`).
- **Always include `description`**: Explain the purpose and expected format.
- **Provide `default` only when sensible**: Configuration options like `environment` or `aws_region` may have defaults, but sensitive values must not have hardcoded defaults.
- **Mark secrets as `sensitive = true`**: Password, token, and secret variables must be marked sensitive.

Example `variables.tf`:
```hcl
variable "aws_region" {
  type        = string
  description = "The AWS region where resources will be provisioned."
  default     = "us-east-1"
}

variable "environment" {
  type        = string
  description = "Deployment environment name (e.g., dev, staging, prod)."
  default     = "production"

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, production."
  }
}

variable "instance_type" {
  type        = string
  description = "EC2 instance size for the application cluster."
  default     = "t3.medium"
}

variable "vpc_cidr" {
  type        = string
  description = "Base CIDR block for the Virtual Private Cloud."
  default     = "10.0.0.0/16"
}
```

#### `locals.tf` Rules:
Use `locals` for values derived internally, environment prefixes, and common tags:
```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

#### `terraform.tfvars.example`
Always create an example variable values file so teammates know how to supply values:
```hcl
aws_region   = "us-east-1"
environment  = "production"
project_name = "checkout-service"
```

### 4. Phase 4: Replace Static Values with Dynamic References (Zero-Drift Guardrails)

Convert static cloud IDs and disjoint resource references into native HCL expressions while protecting live infrastructure from accidental replacement:

#### Immutability & `ForceNew` Guardrail (Critical for Brownfield / Terraformer)
- **Distinguish Terraform addresses from cloud `name` attributes**: Renaming a Terraform resource label (`tfer--prod-main-vpc` → `main`) with a `moved {}` block is safe and zero-downtime. Changing a cloud `name` attribute (e.g., `google_compute_network.name`, `google_compute_subnetwork.name`, `google_compute_firewall.name`, `google_compute_instance.name`, or `aws_s3_bucket.bucket`) is **`ForceNew`** and **destroys and recreates** live infrastructure. Parameterize existing cloud names in `variables.tf` / `terraform.tfvars` instead of rewriting them to `"${local.name_prefix}-..."`.
- **Pin imported images/AMIs on brownfield imports**: Switching a hardcoded `ami-*` or `debian-12-bookworm-v20240515` to a `most_recent = true` / family `data` lookup will replace existing VMs if a newer image exists. Pin the exact imported image/AMI name or ID via a variable/`data` filter when zero-drift is required, and only enable `most_recent = true` when the user opts into image upgrades.
- **Keep security hardening separate from zero-drift refactoring**: Do not silently change behavioral/security attributes (e.g., `http_tokens` `"optional"` → `"required"`, `public_access_prevention` `"inherited"` → `"enforced"`, or VM OAuth `scopes`) during a zero-drift migration. Treat hardening as an explicit opt-in or clearly list every intentional diff.
- **Preserve legacy tag/label keys when migrating**: If existing resources use `env = "prod"`, preserve existing keys (or add standardized keys alongside them) unless replacing keys is approved.

#### Terraformer-Specific Cleanup Checklist
- Strip computed read-only state attributes (`tags_all` on AWS; `effective_labels` and `terraform_labels` on GCP) and empty/default attributes (`""`, `[]`, `{}`, `false`, `0`).
- Remove conflicting state-dumped sub-blocks (e.g., EC2 primary `network_interface` when `subnet_id` is set).
- Replace legacy `${data.terraform_remote_state.*.outputs.*}` interpolations and full self-link URLs with direct resource references (e.g., `google_compute_network.main.id`) or `data` lookups.
- Collapse numbered duplicate resources (`vm-01`, `vm-02`) into a single `for_each` resource paired with `moved` blocks (`to = google_compute_instance.api["01"]`).
- Extract inline `\n`-escaped `user_data` / `startup-script` strings into `scripts/` files loaded via `file()` or `templatefile()`.
- When mutating GCP VM `scopes` or `machine_type`, ensure `allow_stopping_for_update = true` is set if in-place updates are intended.

#### Anti-pattern (Raw / Hardcoded):
```hcl
# BAD: Hardcoded IDs and literal strings
resource "aws_instance" "tfer--i-0123456789abcdef0" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  subnet_id     = "subnet-01234567"
  vpc_security_group_ids = [
    "sg-0abcdef1234567890"
  ]
}
```

#### Standard Pattern (Refactored):
```hcl
# 1. State preservation: Tell Terraform this resource was renamed (prevents destroy & recreate)
moved {
  from = aws_instance.tfer--i-0123456789abcdef0
  to   = aws_instance.web
}

# 2. Dynamic lookup for base image (pin exact name for zero-drift brownfield, or most_recent for greenfield)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = [var.ami_name_pattern]
  }
}

# 3. Standardized resource definition with dynamic references
resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.private[0].id
  vpc_security_group_ids = [aws_security_group.web.id]

  # Omit common_tags here if provider.tf already sets default_tags
  tags = {
    Name = var.instance_name
  }
}
```

### 5. Phase 5: Formatting, Drift Audit, and Validation

1. **Pre-Plan Drift Audit Checklist** (always perform, especially if `terraform` CLI is not installed locally):
   - Compare every resource's cloud `name`, `image`/`ami`, `location`/`zone`, `tags`/`labels`, and security/network settings against the imported source.
   - Verify no `ForceNew` fields were accidentally changed and that every `tfer--*` resource has a matching `moved {}` block.
   - Separate and report any **intentional** diffs (e.g., security hardening) from zero-drift refactoring.

2. **Run Terraform CLI checks** (when installed):
```bash
# 1. Format code to standard canonical HCL indentation
terraform fmt -recursive

# 2. Initialize provider plugins (use -backend=false when validating offline/without state) and validate syntax
terraform init -backend=false
terraform validate

# 3. If working in an active environment with state, ensure plan produces NO unintended diffs
terraform plan
```

3. **Summarize `terraform plan` Changes (when any diff is detected)**:
   If `terraform plan` reports anything other than `No changes`, **stop before recommending `apply`** and present a structured summary:
   - **Plan totals**: State the exact resource counts (`X to add, Y to change, Z to destroy`) and any `moved` state transitions.
   - **High-risk changes first (`-/+` replace, `-` destroy)**: Highlight every resource marked for replacement or deletion, name the `ForceNew` attribute causing it (e.g., `name`, `image`, `boot_disk`), and explain how to fix it (such as adding a missing `moved {}` block or restoring the original value in `tfvars`).
   - **In-place updates (`~`) & creations (`+`)**: List each affected resource and the exact attributes changing (e.g., `labels`, `tags`, `scopes`, `public_access_prevention`).
   - **Intentional vs. unintended drift**: Clearly mark which diffs are intentional (opt-in security hardening or tag standardization) versus unintended brownfield drift so the user can approve or revert them.

---

## Detailed Guidelines & References

- Refer to [Standard File Layout](references/standard_file_layout.md) for domain grouping patterns.
- Refer to [Refactoring Rules](references/refactoring_rules.md) for naming, `ForceNew` guardrails, tags/labels, and anti-pattern checklists.
