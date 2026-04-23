# 07 - State Management

> 📊 **Exam Weight:** 12%

---

## What is Terraform State?

Terraform state is a file that maps real-world resources to your configuration and tracks metadata.

### Purpose of State

| Purpose | Description |
|---------|-------------|
| **Resource Mapping** | Maps resource addresses to real-world IDs |
| **Metadata Storage** | Stores resource attributes and dependencies |
| **Performance** | Caches resource attributes to avoid API calls |
| **Collaboration** | Synchronizes work across teams |

### State File Contents

```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 15,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "outputs": {
    "instance_id": {
      "value": "i-1234567890abcdef0",
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
            "ami": "ami-0c55b159cbfafe1f0",
            "id": "i-1234567890abcdef0",
            "instance_type": "t2.micro",
            "tags": {
              "Name": "WebServer"
            }
          }
        }
      ]
    }
  ]
}
```

---

## Local State

By default, Terraform stores state locally in `terraform.tfstate`.

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfstate          # State file
└── terraform.tfstate.backup   # Backup (previous state)
```

### ⚠️ Security Warning

**State files contain sensitive data!**
- Passwords
- API keys
- Private keys
- Connection strings

**Never commit state files to version control!**

```bash
# Add to .gitignore
echo '*.tfstate*' >> .gitignore
echo '.terraform/' >> .gitignore
```

---

## Remote State

For teams, use remote state backends.

### Backend Types

| Backend | Best For |
|---------|----------|
| **S3** | AWS environments |
| **Azure Blob** | Azure environments |
| **GCS** | GCP environments |
| **Terraform Cloud** | Multi-cloud, team collaboration |
| **Consul** | HashiCorp ecosystem |
| **PostgreSQL** | Generic database backend |
| **etcd** | Kubernetes environments |

### S3 Backend Example

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "production/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

**Prerequisites:**
- S3 bucket (versioning recommended)
- DynamoDB table for state locking

### Terraform Cloud Backend

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

---

## State Locking

Prevents concurrent state modifications.

### How Locking Works

```
User A: terraform apply
   └─> Lock acquired
   └─> Making changes
   └─> Lock released

User B: terraform apply (during User A's changes)
   └─> Error: State is already locked
```

### Backend Support

| Backend | Locking Support |
|---------|-----------------|
| S3 | Yes (via DynamoDB) |
| Terraform Cloud | Yes (built-in) |
| Azure Blob | Yes |
| GCS | Yes |
| PostgreSQL | Yes |
| Consul | Yes |
| Local | No |

### DynamoDB Table for S3 Locking

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

---

## State Commands

### View State

```bash
# List all resources
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Show full state
terraform show

# Pull raw state
terraform state pull
```

### Modify State

```bash
# Move resource to new address
terraform state mv aws_instance.web aws_instance.new_web

# Remove resource from state (does NOT destroy!)
terraform state rm aws_instance.web

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0

# Push state (use with caution!)
terraform state push custom.tfstate
```

### State Subcommands

| Command | Purpose |
|---------|---------|
| `list` | List resources in state |
| `show` | Show resource details |
| `mv` | Move/rename resource |
| `rm` | Remove resource from state |
| `pull` | Pull remote state |
| `push` | Push state (dangerous!) |

---

## State Operations

### Refresh

Updates state file with real-world status.

```bash
# Refresh state (legacy command)
terraform refresh

# Modern approach: refresh-only plan
terraform apply -refresh-only
```

**When to use:**
- Manual changes made outside Terraform
- Detect drift

### Import

Bring existing resources under Terraform management.

```bash
# Import syntax
terraform import <resource_type>.<name> <resource_id>

# Example: Import EC2 instance
terraform import aws_instance.web i-1234567890abcdef0

# Example: Import S3 bucket
terraform import aws_s3_bucket.my_bucket my-bucket-name
```

**Import Process:**
1. Write configuration matching the resource
2. Run `terraform import`
3. Verify with `terraform plan` (should show no changes)

### Tainting / Replacing

Mark a resource for recreation.

```bash
# Old way (deprecated)
terraform taint aws_instance.web
terraform apply

# New way (Terraform 0.15.2+)
terraform apply -replace=aws_instance.web
```

---

## State Best Practices

### 1. Use Remote State for Teams

```hcl
# Never use local state in production
terraform {
  backend "s3" {
    bucket = "company-terraform-state"
    key    = "project/terraform.tfstate"
    region = "us-east-1"
    encrypt = true
    dynamodb_table = "terraform-locks"
  }
}
```

### 2. Enable State Encryption

- S3: `encrypt = true`
- Azure: Storage Service Encryption
- GCS: Encryption at rest (default)

### 3. Enable State Versioning

```bash
# Enable S3 versioning
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled
```

### 4. Backup State

```bash
# Backup before risky operations
cp terraform.tfstate terraform.tfstate.backup.$(date +%Y%m%d)
```

### 5. State Isolation

Use separate state files for different environments:

```
terraform/
├── dev/
│   └── main.tf
├── staging/
│   └── main.tf
└── prod/
    └── main.tf
```

---

## Sensitive Data in State

### The Problem

```json
{
  "resources": [
    {
      "type": "aws_db_instance",
      "instances": [
        {
          "attributes": {
            "password": "SuperSecret123!"  # ⚠️ Plain text!
          }
        }
      ]
    }
  ]
}
```

### Solutions

1. **Use remote state with encryption**
2. **Use a secrets manager** (AWS Secrets Manager, Vault)
3. **Mark outputs as sensitive**
   ```hcl
   output "db_password" {
     value     = aws_db_instance.main.password
     sensitive = true
   }
   ```
4. **Use `nonsensitive()` function carefully**

---

## State Workspaces

Workspaces maintain separate state files for the same configuration.

```bash
# Create workspace
terraform workspace new production

# List workspaces
terraform workspace list

# Select workspace
terraform workspace select production

# Current workspace
terraform workspace show

# Delete workspace
terraform workspace delete staging
```

### Workspace State Files

```
# Local
terraform.tfstate.d/
├── production/
│   └── terraform.tfstate
├── development/
│   └── terraform.tfstate
└── default/
    └── terraform.tfstate

# S3 Backend
s3://bucket/
├── env:/
│   ├── production/
│   │   └── project/terraform.tfstate
│   └── development/
│       └── project/terraform.tfstate
```

---

## Exam Tips

1. **State purpose** - Maps config to real resources, tracks metadata
2. **Sensitive data** - State files contain secrets; secure them!
3. **Remote state** - Required for team collaboration
4. **State locking** - Prevents concurrent modifications
5. **Import** - Brings existing resources under Terraform control
6. **Workspaces** - Separate state for same config (not for environments)
7. **State commands** - Know `list`, `show`, `mv`, `rm`, `pull`

---

[⬅️ Previous: Workflow](06-workflow.md) | [Next: Configurations ➡️](08-configurations.md)
