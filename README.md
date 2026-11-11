# Terraform Standardize

An AI agent skill and Antigravity plugin for refactoring raw, brownfield, and [Terraformer](https://github.com/GoogleCloudPlatform/terraformer)-imported Terraform configurations (AWS, GCP, Azure) into clean, modular, zero-drift HashiCorp standards.

## What It Does

1. **Audit & CLI Detection**: Detects `terraform` / `tofu` availability, missing core files, hardcoded cloud IDs/ARNs/self-links, and formatting baselines.
2. **Standard Multi-File Architecture**: Splits monolithic `.tf` files into `versions.tf`, `provider.tf`, `backend.tf`, `variables.tf`, `locals.tf`, `data.tf`, `outputs.tf`, `terraform.tfvars.example`, `scripts/`, and domain files (`networking.tf`, `security.tf`, `compute.tf`, `storage.tf`).
3. **Parameterization & Tagging**: Extracts literals into typed variables with descriptions/validation and establishes a single source of truth for tags/labels without duplicating provider defaults.
4. **Zero-Drift & `ForceNew` Guardrails**:
   - Generates `moved {}` blocks when renaming `tfer--*` Terraform addresses.
   - Preserves `ForceNew` cloud `name` attributes and pins imported AMI/image versions so existing infrastructure is not destroyed and recreated.
   - Strips read-only state attributes (`tags_all`, `effective_labels`, `terraform_labels`) and collapses numbered duplicates into `for_each`.
   - Extracts inline `\n`-escaped startup scripts into `scripts/`.
5. **Formatting, Validation & Plan Summaries**: Runs `terraform fmt -recursive`, `terraform init -backend=false`, `terraform validate`, and `terraform plan` (summarizing any `+`, `~`, `-/+`, or `moved` diffs before applying).

## Installation

### Option 1: Cross-Agent CLI (`skills`)

Works with Antigravity, Claude Code, Codex, Cursor, and Gemini CLI:

```bash
npx skills add daily-dreamer/terraform-standardize
```

### Option 2: Antigravity Plugin

Clone this repository directly into your global plugins folder:

- **Windows (PowerShell)**:
  ```powershell
  git clone https://github.com/daily-dreamer/terraform-standardize.git "$env:USERPROFILE\.gemini\config\plugins\terraform-standardize"
  ```
- **macOS / Linux**:
  ```bash
  git clone https://github.com/daily-dreamer/terraform-standardize.git ~/.gemini/config/plugins/terraform-standardize
  ```

### Option 3: Project-Local Skill

Copy `skills/terraform-standardize/` into your repository's `.agents/skills/` folder:

```bash
cp -r skills/terraform-standardize /path/to/your-repo/.agents/skills/
```

## Example Prompts

- *"Standardize this Terraformer-imported directory with zero drift."*
- *"Refactor this monolithic `main.tf` into standard Terraform files with typed variables."*
- *"Audit these imported `.tf` files for hardcoded IDs and `ForceNew` replacement risks."*

## Repository Structure

```text
terraform-standardize/
├── plugin.json
├── README.md
├── LICENSE
├── assets/
│   └── logo.svg
└── skills/
    └── terraform-standardize/
        ├── SKILL.md
        └── references/
            ├── refactoring_rules.md
            └── standard_file_layout.md
```

## License

Licensed under the [Apache License 2.0](LICENSE).
