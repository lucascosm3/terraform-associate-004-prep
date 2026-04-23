# 04 - Terraform CLI Commands

> 📊 **Exam Weight:** 12%

---

## Essential Commands Overview

```
terraform init          # Initialize working directory
terraform plan          # Preview changes
terraform apply         # Apply changes
terraform destroy       # Destroy infrastructure
terraform validate      # Validate configuration
terraform fmt           # Format configuration files
```

---

## Initialization Commands

### `terraform init`

Initializes a working directory containing Terraform configuration files.

```bash
# Basic initialization
terraform init

# Initialize and upgrade providers
terraform init -upgrade

# Initialize with specific backend config
terraform init -backend-config="backend.hcl"

# Initialize and migrate state (reconfigure backend)
terraform init -reconfigure

# Initialize and migrate state to new backend
terraform init -migrate-state
```

**What init does:**
- Downloads required providers
- Sets up backend configuration
- Downloads modules
- Creates `.terraform` directory

---

## Planning Commands

### `terraform plan`

Creates an execution plan, showing what actions Terraform will take.

```bash
# Basic plan
terraform plan

# Save plan to file
terraform plan -out=tfplan

# Plan and specify variables
terraform plan -var="instance_type=t2.large"
terraform plan -var-file="production.tfvars"

# Plan destroy
terraform plan -destroy

# Plan with detailed exit codes
terraform plan -detailed-exitcode
# Exit codes:
# 0 - Succeeded, empty diff (no changes)
# 1 - Error
# 2 - Succeeded, non-empty diff (changes present)

# Refresh state before plan
terraform plan -refresh=true

# Skip refresh for faster planning
terraform plan -refresh=false
```

### `terraform show`

Displays a saved plan or state file.

```bash
# Show saved plan
terraform show tfplan

# Show state file
terraform show

# Show in JSON format
terraform show -json
```

---

## Apply Commands

### `terraform apply`

Applies the changes required to reach the desired state.

```bash
# Basic apply (interactive confirmation)
terraform apply

# Apply without confirmation
terraform apply -auto-approve

# Apply saved plan
terraform apply tfplan

# Apply with variables
terraform apply -var="instance_type=t2.large"
terraform apply -var-file="production.tfvars"

# Apply specific target
terraform apply -target=aws_instance.web

# Apply with parallelism (default: 10)
terraform apply -parallelism=5

# Apply and replace specific resource
terraform apply -replace=aws_instance.web
```

### `terraform apply -replace` vs `taint`

```bash
# Old way (taint is deprecated in favor of -replace)
terraform taint aws_instance.web
terraform apply

# New way (Terraform 0.15.2+)
terraform apply -replace=aws_instance.web
```

---

## Destruction Commands

### `terraform destroy`

Destroys all resources managed by the configuration.

```bash
# Basic destroy
terraform destroy

# Destroy without confirmation
terraform destroy -auto-approve

# Destroy specific target
terraform destroy -target=aws_instance.web

# Create destroy plan
terraform plan -destroy -out=destroy.plan
terraform apply destroy.plan
```

**⚠️ Warning:** `destroy` is irreversible. Always double-check before running!

---

## Validation & Formatting

### `terraform validate`

Validates the configuration files for syntax and internal consistency.

```bash
# Basic validation
terraform validate

# Validate with JSON output
terraform validate -json
```

**When to use:**
- Before committing code
- In CI/CD pipelines
- After making configuration changes

### `terraform fmt`

Rewrites configuration files to canonical format and style.

```bash
# Format current directory
terraform fmt

# Format recursively
terraform fmt -recursive

# Check if files are formatted (CI/CD)
terraform fmt -check

# Show differences
terraform fmt -diff

# Format specific file
terraform fmt main.tf
```

**Best Practice:** Always run `terraform fmt` before committing changes!

---

## State Management Commands

### `terraform state`

Subcommands for advanced state management.

```bash
# List resources in state
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Pull raw state
terraform state pull > state.tfstate

# Push state (use with caution!)
terraform state push state.tfstate

# Move resource to different address
terraform state mv aws_instance.web aws_instance.new_web

# Remove resource from state (does NOT destroy!)
terraform state rm aws_instance.web

# Import existing resource to state
terraform import aws_instance.web i-1234567890abcdef0
```

### State Inspection

```bash
# List all resources
terraform state list

# List with pattern
terraform state list 'aws_instance.*'

# Show resource details
terraform state show aws_instance.web

# Pull raw state JSON
terraform state pull
```

---

## Workspace Commands

### `terraform workspace`

Manages multiple state files for the same configuration.

```bash
# List workspaces
terraform workspace list

# Show current workspace
terraform workspace show

# Create new workspace
terraform workspace new production

# Select workspace
terraform workspace select production

# Delete workspace
terraform workspace delete staging
terraform workspace delete -force staging
```

---

## Provider & Module Commands

### `terraform providers`

Shows information about required providers.

```bash
terraform providers
```

### `terraform get`

Downloads and updates modules.

```bash
# Download modules
terraform get

# Update modules
terraform get -update
```

---

## Debugging & Inspection

### `terraform console`

Interactive console for evaluating expressions.

```bash
# Open console
terraform console

# Test expressions
> var.instance_type
t2.micro

> 1 + 1
2

> aws_instance.web.id
"i-1234567890abcdef0"

> exit  # or Ctrl+C
```

### `terraform output`

Reads output values from state file.

```bash
# Show all outputs
terraform output

# Show specific output
terraform output instance_id

# Show raw output (no quotes)
terraform output -raw instance_id

# Show JSON format
terraform output -json
```

---

## Graph Commands

### `terraform graph`

Generates a visual representation of the configuration or execution plan.

```bash
# Generate DOT format
terraform graph

# Generate and visualize (requires Graphviz)
terraform graph | dot -Tpng > graph.png

# Graph of plan
terraform graph -type=plan
```

---

## Version & Environment

### `terraform version`

Shows current Terraform version.

```bash
terraform version
terraform -v
```

### `terraform providers`

Lists providers required by the configuration.

```bash
terraform providers
terraform providers -json
```

---

## Command Quick Reference

| Command | Purpose |
|---------|---------|
| `init` | Initialize working directory |
| `validate` | Validate configuration syntax |
| `plan` | Preview changes |
| `apply` | Apply changes |
| `destroy` | Destroy infrastructure |
| `fmt` | Format configuration |
| `show` | Show state or plan |
| `state` | State management |
| `workspace` | Workspace management |
| `output` | Show outputs |
| `console` | Interactive console |
| `graph` | Generate dependency graph |

---

## Exam Tips

1. **Exit codes** - `plan -detailed-exitcode`: 0=no changes, 1=error, 2=changes pending
2. **Order matters** - Always `init` → `validate` → `plan` → `apply`
3. **State vs Reality** - `refresh` updates state; doesn't change infrastructure
4. **Targeted operations** - Use `-target` sparingly; can cause dependency issues
5. **Format check** - Use `fmt -check` in CI/CD pipelines
6. **Replace vs Taint** - `-replace` is the modern replacement for `taint`

---

[⬅️ Previous: Terraform Basics](03-terraform-basics.md) | [Next: Modules ➡️](05-modules.md)
