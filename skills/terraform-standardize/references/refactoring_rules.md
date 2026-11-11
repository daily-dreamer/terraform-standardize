# Terraform Refactoring Rules & Anti-Patterns

Follow these practical transformation rules when refactoring raw, brownfield, or imported Terraform code.

---

## 1. Naming Conventions

### Resource Labels
- **Use `snake_case`**: All resource labels and variable names must use lowercase letters, numbers, and underscores (`_`).
- **Do not repeat resource types in labels**:
  - **Bad**: `resource "aws_security_group" "web_sg"` (redundant `_sg`)
  - **Good**: `resource "aws_security_group" "web"`
  - **Bad**: `resource "aws_vpc" "vpc_main"` (redundant `vpc_`)
  - **Good**: `resource "aws_vpc" "main"`
- **Singular vs Plural**:
  - Single instance/resource: singular (e.g., `resource "aws_instance" "web"`)
  - Loop or multi-resource count/for_each: descriptive plural or role (e.g., `resource "aws_subnet" "private"`)

---

## 2. Hardcoded Values to Parameterized Variables

### Never Leave Plain Strings for:
| String Pattern | Refactoring Target |
| :--- | :--- |
| `"us-east-1"`, `"us-central1"` | `var.aws_region` / `var.region` |
| `"prod"`, `"production"`, `"staging"` | `var.environment` |
| `"10.0.0.0/16"`, CIDR ranges | `var.vpc_cidr` or `var.subnet_cidr` |
| `"t3.micro"`, `"e2-standard-4"` | `var.instance_type` / `var.machine_type` |
| `"ami-0123456789abcdef0"`, `"debian-12-..."` | `data.aws_ami` / `data.google_compute_image` (pin exact image name for brownfield zero-drift) |
| `"subnet-0abc123"`, `"vpc-0123"`, full GCP self-links | Dynamic reference: `aws_subnet.private[count.index].id` / `google_compute_network.main.id` |
| `"sg-0987654"` | Dynamic reference: `aws_security_group.app.id` |
| `"my-company-prod-bucket"` | `var.bucket_name` (greenfield: `"${local.name_prefix}-data-bucket"`) |

> **ForceNew Warning**: Do not alter existing cloud `name` attributes (VPCs, subnets, firewalls, VMs, buckets) during brownfield standardization unless recreation is explicitly approved—use `variables.tf` / `terraform.tfvars` to preserve the live resource names while refactoring Terraform state labels via `moved {}`.

---

## 3. Dynamic References vs. Static IDs

When cloud resources depend on one another, never hardcode the cloud resource ID, ARN, or self-link. Link their Terraform attributes:

### Anti-Pattern:
```hcl
resource "aws_instance" "app" {
  subnet_id              = "subnet-08123982348"
  vpc_security_group_ids = ["sg-03482394823"]
}
```

### Standard Pattern:
```hcl
resource "aws_instance" "app" {
  subnet_id              = aws_subnet.app[0].id
  vpc_security_group_ids = [aws_security_group.app.id]
}
```

---

## 4. Tagging & Labeling Standards

Use `locals.tf` to define a single source of truth for resource tags (AWS/Azure) or labels (GCP):

```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Repository  = var.repository_url
  }
}
```

Choose **one** application strategy (never combine both redundantly):

1. **Provider-level defaults (Recommended)**: Configure `default_tags` (AWS) or `default_labels` (GCP `>= 5.5`) in `provider.tf`, and specify only resource-specific keys (such as `Name` or `purpose`) on individual resources:
```hcl
# provider.tf
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = local.common_tags
  }
}

# compute.tf
resource "aws_instance" "app" {
  # ...
  tags = {
    Name = var.instance_name
    Role = "application-server"
  }
}
```

2. **Explicit `merge()` per resource**: Omit `default_tags`/`default_labels` from `provider.tf` and merge `local.common_tags` / `local.common_labels` on each resource:
```hcl
resource "aws_instance" "app" {
  # ...
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-app"
    Role = "application-server"
  })
}
```

> **Brownfield Tag/Label Preservation**: Strip read-only state-dumped fields (`tags_all`, `effective_labels`, `terraform_labels`), and preserve legacy keys (e.g., `env` vs `environment`) when zero-drift against imported state is required.
