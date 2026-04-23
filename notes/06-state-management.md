# 06 - Domain 6: Terraform State Management

> Exam Weight: ~13% | Tested on Terraform 1.12

---

## Objectives

- [x] **6a:** Describe the local backend
- [x] **6b:** Describe state locking
- [x] **6c:** Configure remote state using the backend block
- [x] **6d:** Manage resource drift and Terraform state (moved block, removed block, refresh-only mode, resource drift)

---

## 6a. The Local Backend

### Default Behavior

By default, Terraform uses the **local backend**, storing state in a file named `terraform.tfstate` in the working directory.

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfstate          # Current state
└── terraform.tfstate.backup   # Previous state (automatic backup)
```

### Local State Characteristics

| Aspect | Description |
|--------|-------------|
| **File location** | `terraform.tfstate` in working directory |
| **Backup** | `terraform.tfstate.backup` (auto-created on each apply) |
| **Locking** | Not supported (local filesystem) |
| **Collaboration** | Not suitable for teams |
| **Security** | File on disk; contains sensitive data |
| **Performance** | Fast (no network calls) |

### Security Warning

**State files contain sensitive data** — passwords, API keys, connection strings, and private keys are stored in plain text.

```bash
# NEVER commit state files to version control
echo '*.tfstate' >> .gitignore
echo '*.tfstate.*' >> .gitignore
echo '.terraform/' >> .gitignore

# Consider committing .terraform.lock.hcl (for reproducibility)
```

### State File Structure

```json
{
  "version": 4,
  "terraform_version": "1.12.0",
  "serial": 42,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "outputs": {
    "instance_ip": {
      "value": "54.123.45.67",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-1234567890abcdef0",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t2.micro",
            "tags": { "Name": "WebServer" }
          }
        }
      ]
    }
  ]
}
```

**Key fields:**
- `version` — State file format version (currently 4)
- `serial` — Incremented on each state update; used for optimistic locking
- `lineage` — Unique identifier; prevents mixing state files from different configurations

### When Local State Is Appropriate

- Individual learning and experimentation
- Prototyping and quick POCs
- Single-person projects with no collaboration needs

**For any team or production environment, use remote state.**

---

## 6b. State Locking

### What Is State Locking?

State locking prevents concurrent operations from corrupting the state file. When one user runs `terraform apply`, Terraform acquires a lock, and other users attempting to run operations will receive an error until the lock is released.

```
User A: terraform apply
   ├─> Lock acquired
   ├─> Making changes
   ├─> State updated
   └─> Lock released

User B: terraform apply (during User A's operation)
   └─> Error: state is locked by another operation
```

### Backend Locking Support

| Backend | Locking Support |
|---------|-----------------|
| **Local** | No (filesystem operations only) |
| **S3 + DynamoDB** | Yes (via DynamoDB) |
| **HCP Terraform** | Yes (built-in) |
| **Azure Blob** | Yes |
| **GCS** | Yes |
| **PostgreSQL** | Yes |
| **Consul** | Yes |

### DynamoDB for S3 State Locking

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Lock Error Handling

```bash
# If a lock is stuck (e.g., after a crash)
terraform force-unlock <LOCK_ID>

# Example
terraform force-unlock 12345-abcde-67890
```

**Warning:** Only use `force-unlock` if you are certain no other Terraform operation is running. It can cause state corruption.

---

## 6c. Configure Remote State Using the Backend Block

### Backend Configuration

The `backend` block configures where Terraform stores state remotely.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Backend Types

| Backend | Use Case | Locking |
|---------|----------|---------|
| **S3** | AWS environments | Yes (with DynamoDB) |
| **Azure Blob** | Azure environments | Yes |
| **GCS** | GCP environments | Yes |
| **HCP Terraform** | Multi-cloud, teams | Yes (built-in) |
| **Consul** | HashiCorp ecosystem | Yes |
| **PostgreSQL** | Generic database | Yes |
| **Local** | Development only | No |

### S3 Backend (AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/abcd1234"
  }
}
```

**Prerequisites:**
- S3 bucket (with versioning enabled)
- DynamoDB table for state locking
- IAM permissions for Terraform

```bash
# Create S3 bucket
aws s3api create-bucket --bucket my-terraform-state-bucket --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket my-terraform-state-bucket \
  --server-side-encryption-configuration file://encryption.json
```

### Azure Blob Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstatestorage"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### GCS Backend (GCP)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "prod/terraform"
  }
}
```

### HCP Terraform Backend

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}
```

### Partial Backend Configuration

```hcl
# Main configuration
terraform {
  backend "s3" {}
}
```

```bash
# Pass backend config via command line
terraform init \
  -backend-config="bucket=my-terraform-state" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=us-east-1"
```

```hcl
# Or via a separate file (backend.hcl)
bucket         = "my-terraform-state"
key            = "prod/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-locks"
```

```bash
terraform init -backend-config=backend.hcl
```

### State Migration

```bash
# Migrate state to a new backend (interactive)
terraform init -migrate-state

# Migrate state without prompts (CI/CD)
terraform init -migrate-state -input=false -no-color

# Reconfigure backend WITHOUT migrating state
terraform init -reconfigure
```

### Backend Best Practices

1. **Always use remote state for teams** — local state is not suitable for collaboration
2. **Enable state locking** — prevents corruption from concurrent operations
3. **Enable encryption** — `encrypt = true` for S3, or server-side encryption
4. **Enable versioning** — allows rollback of state
5. **Use separate state files per environment** — dev, staging, prod should have separate states
6. **Never commit state to version control** — add `.tfstate` to `.gitignore`

---

## 6d. Resource Drift and State Management

### What Is Resource Drift?

Resource drift occurs when the actual state of infrastructure differs from what Terraform recorded in state. This can happen when:
- Resources are modified manually (through the console or CLI)
- External processes change resources
- Resources are deleted outside of Terraform

### Detecting Drift

```bash
# Plan shows drift
terraform plan
# Example output:
# ~ aws_instance.web
#     instance_type: "t2.micro" => "t2.large"  # Changed outside Terraform!
```

```bash
# Detailed exit codes
terraform plan -detailed-exitcode
# Exit 0 = no changes (no drift)
# Exit 1 = error
# Exit 2 = changes needed (drift detected)
```

### Refresh-Only Mode (Replaces `terraform refresh`)

**Important:** `terraform refresh` is superseded by `terraform apply -refresh-only`.

The refresh-only mode updates the state file to match real infrastructure without making any configuration changes.

```bash
# Update state to match real infrastructure (no config changes)
terraform apply -refresh-only

# Or use plan to preview what would be refreshed
terraform plan -refresh-only
```

**When to use refresh-only:**
- When you suspect drift and want to see what changed
- Before running a normal plan to ensure state is up to date
- After manual changes were made outside Terraform

```bash
# Step 1: Refresh state
terraform apply -refresh-only

# Step 2: Review changes and apply
terraform plan
terraform apply
```

### The `moved` Block (Terraform 1.1+)

The `moved` block tells Terraform that a resource has been renamed or moved without destroying and recreating it.

```hcl
# Old resource address (removed from configuration)
# resource "aws_instance" "old_web" { ... }

# New resource address (in configuration)
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Moved block — no resource destruction or recreation
moved {
  from = aws_instance.old_web
  to   = aws_instance.web
}
```

**Multiple moved blocks:**

```hcl
resource "aws_security_group" "web_sg" {
  name = "web-security-group"
}

resource "aws_subnet" "app_subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
}

moved {
  from = aws_security_group.web
  to   = aws_security_group.web_sg
}

moved {
  from = aws_subnet.app
  to   = aws_subnet.app_subnet
}
```

**Moved block in modules:**

```hcl
moved {
  from = module.old_vpc
  to   = module.vpc
}
```

**Key points:**
- The `moved` block updates the state address without destroying resources
- After the first apply with the `moved` block, it can be removed or kept
- It works across module boundaries

### The `removed` Block (Terraform 1.7+)

The `removed` block removes a resource from Terraform management WITHOUT destroying the real infrastructure.

```hcl
# Resource still referenced in config but will be unmanaged
removed {
  from = aws_instance.deprecated

  lifecycle {
    destroy = false  # Do NOT destroy the real resource
  }
}
```

**Common use cases:**
- Removing a resource from Terraform management that should continue to exist
- Migrating resources to a different Terraform configuration
- Decommissioning Terraform management of a resource

```hcl
# Remove from module management
removed {
  from = module.old_networking
  lifecycle {
    destroy = false
  }
}
```

**`destroy = true` vs `destroy = false`:**

| Value | Behavior |
|-------|----------|
| `destroy = false` | Remove from state only; real resource is preserved |
| `destroy = true` | Remove from state AND destroy the real resource |

### State Manipulation Commands

```bash
# List all resources in state
terraform state list

# List resources matching a pattern
terraform state list 'aws_instance.*'
terraform state list 'module.vpc.*'

# Show details of a specific resource
terraform state show aws_instance.web

# Pull remote state to local file
terraform state pull > backup.tfstate

# Push local state to remote (use with extreme caution!)
terraform state push backup.tfstate

# Move a resource within state (refactoring)
terraform state mv aws_instance.old aws_instance.new

# Remove a resource from state (does NOT destroy the real resource)
terraform state rm aws_instance.deprecated

# Move resource between modules
terraform state mv aws_instance.web module.app.aws_instance.web
```

### `terraform state mv` for Refactoring

```bash
# Rename a resource in state
terraform state mv aws_instance.old_web aws_instance.new_web

# Move resource to a module
terraform state mv aws_instance.web module.app.aws_instance.web

# Move an entire module
terraform state mv module.old_module module.new_module
```

**Note:** The `moved` block is now the preferred way to handle resource renames, but `terraform state mv` is still useful for interactive refactoring.

### State Best Practices

1. **Use remote state for teams** — never use local state in production
2. **Enable state locking** — prevent concurrent modifications
3. **Enable encryption** — state contains sensitive data
4. **Back up state before risky operations** — `cp terraform.tfstate terraform.tfstate.backup`
5. **Use separate state files per environment** — dev, staging, prod
6. **Use `moved` blocks** for renames — instead of `state mv` for permanent refactoring
7. **Use `removed` blocks** for unmanaging resources — instead of `state rm`
8. **Use `-refresh-only`** instead of the deprecated `refresh` command

---

## Exam Tips

1. **Local state has no locking** — not suitable for teams
2. **State files contain secrets** — always encrypt remote state
3. **S3 backend needs DynamoDB** — for state locking
4. **`terraform force-unlock`** — only use when certain no operation is running
5. **`moved` block** — renames resources without destroy/create
6. **`removed` block with `destroy = false`** — removes from state, keeps real resource
7. **`-refresh-only` replaces `terraform refresh`** — the old command is superseded
8. **Drift detection** — use `terraform plan` or `terraform plan -detailed-exitcode`
9. **State migration** — `terraform init -migrate-state` to move state to a new backend
10. **`.terraform.lock.hcl`** should be committed; `.terraform/` and `.tfstate` should not

---

## Quick Review Questions

1. **Q:** What are the key purposes of Terraform state?
   **A:** Resource mapping (config → real resources), metadata storage, performance caching, and dependency tracking.

2. **Q:** Why should you never commit `.tfstate` files to version control?
   **A:** They contain sensitive data (passwords, keys, connection strings) and are not meant to be shared.

3. **Q:** How does state locking work with S3 backends?
   **A:** A DynamoDB table is used to acquire and release locks, preventing concurrent state modifications.

4. **Q:** What command replaces the deprecated `terraform refresh`?
   **A:** `terraform apply -refresh-only` (or `terraform plan -refresh-only` to preview).

5. **Q:** What does a `moved` block do?
   **A:** It tells Terraform that a resource was renamed, updating the state address without destroying/recreating the resource.

6. **Q:** What does a `removed` block with `destroy = false` do?
   **A:** It removes the resource from Terraform state without destroying the actual infrastructure.

7. **Q:** How do you migrate state from local to remote?
   **A:** Configure the backend block and run `terraform init -migrate-state`.

---

[⬅️ Previous: Modules](05-modules.md) | [Next: Maintain Infrastructure ➡️](07-maintain-infrastructure.md)