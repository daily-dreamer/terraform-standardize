# Terraform Standard File Layout Reference

A clean, predictable directory structure allows teams to navigate, maintain, and review infrastructure code easily.

## Root Module Layout

In a standard Terraform root module, split responsibilities across predictable filenames rather than bundling everything into a single monolithic `main.tf`:

```text
my-infrastructure/
├── versions.tf               # Terraform block & required_providers constraints
├── provider.tf               # Provider configurations (AWS, GCP, Azure)
├── backend.tf                # Remote backend configuration (S3, GCS, AzureRM, Terraform Cloud)
├── variables.tf              # Input variable declarations with types & descriptions
├── terraform.tfvars.example  # Template showing expected variable values
├── locals.tf                 # Local computed values, name prefixes, common tags/labels
├── data.tf                   # External data sources (AMIs/images, VPC lookups, caller identity)
├── scripts/                  # Extracted startup/user-data scripts loaded via file() or templatefile()
├── networking.tf             # VPC, subnets, route tables, gateways (or domain equivalent)
├── compute.tf                # EC2/GCE VMs, Auto Scaling Groups, Launch Templates
├── security.tf               # Security Groups, Cloud Firewalls, IAM roles and policies
├── storage.tf                # S3/GCS buckets, EBS/PD volumes, RDS/Cloud SQL databases
└── outputs.tf                # Module outputs exported for users or remote state
```

---

## File Responsibilities

### `versions.tf` / `provider.tf`
- **`versions.tf`**: Contains the `terraform { required_version = "..." required_providers { ... } }` block.
- **`provider.tf`**: Configures the cloud provider credentials, default region/project, and `default_tags` (AWS) or `default_labels` (GCP `>= 5.5`).

### `variables.tf`
- Only contains `variable` blocks.
- Every variable must specify:
  - `type`: Explicit primitive or structural type constraint.
  - `description`: Human-readable summary of purpose and acceptable values.
  - `default`: Optional; omit for mandatory inputs.
  - `sensitive`: Set to `true` for secrets, passwords, or keys.

### `locals.tf`
- Only contains `locals` blocks.
- Houses derived naming conventions (e.g., `"${var.project}-${var.environment}"`).
- Defines consistent tags map (`local.common_tags`).

### `outputs.tf`
- Only contains `output` blocks.
- Includes `description` for all exported values.
- Marks sensitive outputs (e.g., database connection strings) with `sensitive = true`.

### Domain Files (`compute.tf`, `networking.tf`, `database.tf`, etc.)
- Organize resources by functional domain rather than keeping 1000+ lines in `main.tf`.
- Keep resources that lifecycle together in the same file (e.g. an ALB and its target groups and listener rules).
