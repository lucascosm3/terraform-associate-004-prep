# 07 - Domain 7: Maintain Infrastructure

> Exam Weight: ~5% | Tested on Terraform 1.12

---

## Objectives

- [x] **7a:** Import existing infrastructure into your Terraform workspace
- [x] **7b:** Use the CLI to inspect state (terraform state list, terraform show)
- [x] **7c:** Describe when and how to use verbose logging (TF_LOG, TF_LOG_PATH)

---

## 7a. Import Existing Infrastructure

### Why Import?

When you have infrastructure that was created outside of Terraform (manually, via console, or by another tool), you can import it into Terraform state to manage it going forward.

### `terraform import` Command

```bash
# Syntax
terraform import <RESOURCE_ADDRESS> <RESOURCE_ID>

# Import an EC2 instance
terraform import aws_instance.web i-1234567890abcdef0

# Import an S3 bucket
terraform import aws_s3_bucket.my_bucket my-bucket-name

# Import a VPC
terraform import aws_vpc.main vpc-12345678

# Import to a module
terraform import module.vpc.aws_vpc.main vpc-12345678

# Import with provider alias
terraform import aws_instance.web i-1234567890abcdef0
```

**Key points:**
- The resource address must match a `resource` block already in your configuration
- `terraform import` only imports into state — it does NOT create configuration
- You must write the corresponding `resource` block manually before or after importing
- The imported resource ID format depends on the provider

### Import Workflow

1. **Identify the resource** to import (type and cloud ID)
2. **Write the resource block** in your configuration
3. **Run `terraform import`** to add it to state
4. **Run `terraform plan`** to verify there are no differences
5. **Adjust configuration** until `terraform plan` shows no changes

```bash
# Step 1: Write the resource block
cat >> main.tf << 'EOF'
resource "aws_instance" "imported_web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
EOF

# Step 2: Import into state
terraform import aws_instance.imported_web i-1234567890abcdef0

# Step 3: Verify - this will likely show differences
terraform plan

# Step 4: Adjust configuration to match real infrastructure
# (Update AMI, instance type, tags, etc. based on plan output)

# Step 5: Verify again
terraform plan  # Should show: No changes
```

### Import for Various AWS Resources

```bash
# EC2 instance — use instance ID
terraform import aws_instance.web i-1234567890abcdef0

# S3 bucket — use bucket name
terraform import aws_s3_bucket.data my-data-bucket

# VPC — use VPC ID
terraform import aws_vpc.main vpc-12345678

# Subnet — use subnet ID
terraform import aws_subnet.public subnet-12345678

# Security group — use security group ID
terraform import aws_security_group.web sg-12345678

# IAM role — use role name
terraform import aws_iam_role.lambda_role my-lambda-role

# RDS instance — use instance identifier
terraform import aws_db_instance.main my-db-instance

# DynamoDB table — use table name
terraform import aws_dynamodb_table.users users-table
```

### Import for Various Azure Resources

```bash
# Resource group — use resource ID
terraform import azurerm_resource_group.main /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-rg

# Virtual network — use resource ID
terraform import azurerm_virtual_network.main /subscriptions/.../resourceGroups/my-rg/providers/Microsoft.Network/virtualNetworks/my-vnet

# Storage account — use resource ID
terraform import azurerm_storage_account.main /subscriptions/.../resourceGroups/my-rg/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

### Import Limitations

- **No configuration generation** — import only adds to state; you must write the resource block
- **Not all resources support import** — check provider documentation
- **One resource at a time** — import works on individual resources
- **Complex resources** — may require multiple attributes to be specified
- **Import is idempotent** — re-importing an already-imported resource is safe

### Config-Driven Import (Terrform 1.5+)

Terraform 1.5 introduced the `import` block, allowing you to declare imports in configuration:

```hcl
# Declare an import in configuration
import {
  to = aws_instance.imported_web
  id = "i-1234567890abcdef0"
}

# Write the resource block
resource "aws_instance" "imported_web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

**Benefits of config-driven import:**
- Import declarations are version-controlled
- Can be planned and reviewed
- No need to run a separate `terraform import` command
- Works with `terraform plan` and `terraform apply`

```hcl
# Generate configuration from import (Terraform 1.5+)
import {
  to = aws_vpc.main
  id = "vpc-12345678"
}

# Terraform can generate a scaffold for the resource
# Run: terraform plan -generate-config-out=generated.tf
```

---

## 7b. CLI State Inspection

### `terraform state list`

Lists all resources currently tracked in the state file.

```bash
# List all resources
terraform state list

# Example output:
# aws_instance.web
# aws_internet_gateway.gw
# aws_route_table.public
# aws_security_group.web
# aws_subnet.public
# aws_vpc.main

# Filter by resource type
terraform state list 'aws_instance.*'

# Filter by module
terraform state list 'module.vpc.*'

# Filter by exact name
terraform state list aws_instance.web
```

### `terraform state show`

Displays detailed attributes of a specific resource from state.

```bash
# Show details of a specific resource
terraform state show aws_instance.web

# Example output:
# # aws_instance.web:
# resource "aws_instance" "web" {
#     ami                          = "ami-0c55b159cbfafe1f0"
#     arn                          = "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
#     availability_zone            = "us-east-1a"
#     id                           = "i-1234567890abcdef0"
#     instance_state               = "running"
#     instance_type                = "t2.micro"
#     private_ip                   = "10.0.1.50"
#     public_ip                    = "54.123.45.67"
#     tags                         = {
#         "Name" = "WebServer"
#     }
# }
```

### `terraform show`

Shows the entire state or a saved plan.

```bash
# Show current state (all resources)
terraform show

# Show a saved plan file
terraform show tfplan

# Show state in JSON format
terraform show -json

# Show plan in JSON format
terraform show -json tfplan
```

**`terraform show` vs `terraform state show`:**

| Command | Purpose |
|---------|---------|
| `terraform show` | Shows ALL resources in state, or a saved plan |
| `terraform state show <addr>` | Shows ONE specific resource details |
| `terraform state list` | Lists resource addresses only (no details) |

### `terraform state mv`

Moves a resource from one address to another in state. Used for refactoring.

```bash
# Rename a resource
terraform state mv aws_instance.old_web aws_instance.new_web

# Move a resource into a module
terraform state mv aws_instance.web module.app.aws_instance.web

# Move an entire module
terraform state mv module.old_module module.new_module
```

**Note:** After using `state mv`, you must also update your configuration to match the new addresses. The `moved` block is now the preferred approach for renames.

### `terraform state rm`

Removes a resource from state without destroying the real infrastructure.

```bash
# Remove a single resource from state
terraform state rm aws_instance.deprecated

# Remove a module from state
terraform state rm module.old_networking
```

**Important:** The real resource still exists! It's just no longer managed by Terraform.

### `terraform state pull` and `terraform state push`

```bash
# Pull remote state to local file (for backup)
terraform state pull > backup.tfstate

# Push local state to remote backend (use with extreme caution!)
terraform state push backup.tfstate
```

**Warning:** `terraform state push` overwrites the remote state. Only use it when absolutely necessary.

### Practical Inspection Workflow

```bash
# 1. List all resources to understand current state
terraform state list

# 2. Inspect a specific resource for troubleshooting
terraform state show aws_instance.web

# 3. View the entire state in JSON (for scripting)
terraform show -json | jq '.values.root_module.resources[] | .address'

# 4. Check if a resource exists in state
terraform state list | grep aws_instance
```

---

## 7c. Verbose Logging (TF_LOG, TF_LOG_PATH)

### What Are TF_LOG Variables?

Terraform provides environment variables to enable verbose logging for debugging and troubleshooting.

### TF_LOG - Logging Level

```bash
# Set logging level
export TF_LOG=TRACE    # Most verbose (everything)
export TF_LOG=DEBUG    # Debug information
export TF_LOG=INFO     # General information
export TF_LOG=WARN     # Warnings only
export TF_LOG=ERROR    # Errors only

# Example usage
TF_LOG=DEBUG terraform apply
```

| Level | Description | Use Case |
|-------|-------------|----------|
| `TRACE` | Extremely detailed; shows every API call and response | Deep debugging |
| `DEBUG` | Detailed; shows provider operations and HTTP requests | Debugging provider issues |
| `INFO` | General operational messages | Understanding flow |
| `WARN` | Warning messages only | Monitoring potential issues |
| `ERROR` | Error messages only | Identifying failures |

### TF_LOG_PATH - Log File Output

```bash
# Write logs to a file instead of stderr
export TF_LOG_PATH=/tmp/terraform-debug.log
export TF_LOG=DEBUG

terraform apply

# View the log
cat /tmp/terraform-debug.log
```

### JSON Logging

```bash
# Enable JSON-formatted logs
export TF_LOG=DEBUG
export TF_LOG_JSON=1
export TF_LOG_PATH=/tmp/terraform-debug.json

terraform apply

# Parse JSON logs with jq
cat /tmp/terraform-debug.json | jq '.message'
```

### Common Debugging Scenarios

#### Debugging Provider Issues

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=/tmp/terraform-provider-debug.log
terraform apply 2>&1 | tee /tmp/terraform-output.log
```

#### Debugging Authentication Issues

```bash
TF_LOG=DEBUG terraform plan 2>&1 | grep -i "auth\|credential\|token"
```

#### Debugging State Lock Issues

```bash
TF_LOG=DEBUG terraform apply 2>&1 | grep -i "lock\|dynamo"
```

#### Debugging Variable Resolution

```bash
TF_LOG=TRACE terraform plan 2>&1 | grep -i "variable\|value"
```

### TF_LOG_CORE and TF_LOG_PROVIDER

```bash
# Log only from Terraform core (not providers)
export TF_LOG_CORE=DEBUG

# Log only from provider plugins
export TF_LOG_PROVIDER=TRACE

# Both
export TF_LOG_CORE=INFO
export TF_LOG_PROVIDER=DEBUG
terraform apply
```

### Cleaning Up

```bash
# Unset environment variables when done
unset TF_LOG
unset TF_LOG_PATH
unset TF_LOG_JSON

# Or for a single command
TF_LOG=DEBUG terraform apply
# (Only set for this command, not persistent)
```

### Best Practices for Debug Logging

1. **Use `TF_LOG=DEBUG`** as a starting point — TRACE is often too verbose
2. **Always set `TF_LOG_PATH`** — keeps terminal output clean and allows searching
3. **Remove or unset log variables** after debugging — logs can be large
4. **Sanitize logs** before sharing — they may contain credentials or tokens
5. **Use `TF_LOG_PROVIDER=DEBUG`** to isolate provider-specific issues
6. **Use JSON logging** for programmatic analysis with `jq` or similar tools

### Common Log Messages

```
# Successful provider initialization
2024-01-15T10:30:00.000Z [INFO]  provider: configuring provider: version=5.31.0

# API call details (TRACE level)
2024-01-15T10:30:01.000Z [TRACE] provider.aws: HTTP Request: PUT https://ec2.us-east-1.amazonaws.com/

# State locking
2024-01-15T10:30:02.000Z [DEBUG] provider: acquired state lock

# Variable resolution
2024-01-15T10:30:03.000Z [TRACE] eval: evaluating variable "instance_type"
```

---

## Exam Tips

1. **`terraform import` does NOT create configuration** — you must write the resource block yourself
2. **Import workflow** — Write config → Import → Plan → Adjust → Plan (no changes)
3. **`terraform state list`** — lists resource addresses in state
4. **`terraform state show <addr>`** — shows details of one resource
5. **`terraform show`** — shows entire state or a saved plan
6. **`terraform show -json`** — JSON output for scripting
7. **`terraform state rm`** — removes from state but does NOT destroy the real resource
8. **TF_LOG levels** — TRACE, DEBUG, INFO, WARN, ERROR (from most to least verbose)
9. **TF_LOG_PATH** — writes logs to a file instead of stderr
10. **Config-driven import** (Terraform 1.5+) — use the `import` block in configuration

---

## Quick Review Questions

1. **Q:** What must you do BEFORE running `terraform import`?
   **A:** Write a `resource` block in your configuration that matches the resource you want to import.

2. **Q:** Does `terraform import` create the resource configuration for you?
   **A:** No — it only adds the resource to state. You must write the configuration manually.

3. **Q:** What is the difference between `terraform show` and `terraform state show`?
   **A:** `terraform show` displays the entire state (or a saved plan); `terraform state show <addr>` shows details of one specific resource.

4. **Q:** What does `terraform state rm` do?
   **A:** Removes a resource from state without destroying the actual infrastructure.

5. **Q:** What is the most verbose TF_LOG level?
   **A:** TRACE — it shows every API call and response.

6. **Q:** How do you send Terraform logs to a file?
   **A:** Set `TF_LOG_PATH=/path/to/file.log` (and `TF_LOG=DEBUG` or another level).

7. **Q:** What is config-driven import (Terraform 1.5+)?
   **A:** Using an `import` block in your configuration to declare imports, making them version-controlled and plannable.

---

[⬅️ Previous: State Management](06-state-management.md) | [Next: HCP Terraform ➡️](08-hcp-terraform.md)