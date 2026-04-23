# 02 - Domain 2: Terraform Fundamentals (Providers + State)

> Exam Weight: ~12% | Tested on Terraform 1.12

---

## Objectives

- [x] **2a:** Install and version Terraform providers
- [x] **2b:** Describe how Terraform uses providers
- [x] **2c:** Write Terraform configuration using multiple providers
- [x] **2d:** Explain how Terraform uses and manages state

---

## 2a. Install and Version Terraform Providers

### The `required_providers` Block

Every Terraform configuration should declare the providers it needs, including their source addresses and version constraints.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.0, < 4.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.23.0"
    }
  }

  required_version = ">= 1.5"
}
```

**Key points:**
- The `source` address follows the format `[hostname]/[namespace]/[type]`
- If `hostname` is omitted, it defaults to `registry.terraform.io`
- Official HashiCorp providers use `hashicorp/<name>`
- Community providers use `<namespace>/<name>`

### Version Constraints

| Operator | Meaning | Example | Allowed Versions |
|----------|---------|---------|-----------------|
| `>=` | Greater than or equal | `>= 2.0` | 2.0, 2.5, 3.0, ... |
| `<=` | Less than or equal | `<= 3.0` | ..., 2.5, 3.0 |
| `~>` | Allow patch updates | `~> 2.0` | 2.0, 2.1, 2.9 (not 3.0) |
| `~>` | Allow minor updates | `~> 2.0.0` | 2.0.0, 2.0.5 (not 2.1.0) |
| `>=, <` | Range | `>= 2.0, < 3.0` | 2.0, 2.5, 2.99 |
| `=` | Exact version | `= 2.3.1` | 2.3.1 only |

**Best practice:** Use `~>` for provider versions to allow patch/minor updates while preventing breaking changes.

### The Dependency Lock File (`.terraform.lock.hcl`)

When you run `terraform init`, Terraform creates or updates `.terraform.lock.hcl` to record the exact versions of providers that were selected.

```hcl
# .terraform.lock.hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:R+FHlJfanPUmr3QFPxCqAt...',
    "zh:086d3cb18e755...",
  ]
}
```

**Key points:**
- **Commit this file to version control** — it ensures all team members and CI use the same provider versions
- It locks the exact provider version, not just the constraint range
- To update locked versions, use `terraform init -upgrade`
- Without the lock file, `terraform init` selects the newest version matching constraints

### `terraform init` Behavior

```bash
# First init — downloads providers and creates lock file
terraform init

# Re-init after adding a new provider — downloads only new provider
terraform init

# Force upgrade of providers beyond lock file
terraform init -upgrade

# Use a local mirror or filesystem provider
terraform init -plugin-dir=/usr/local/share/terraform/plugins

# Reconfigure backend without migrating state
terraform init -reconfigure
```

**What `terraform init` does:**
1. Downloads provider plugins to `.terraform/providers/`
2. Downloads module sources
3. Initializes the backend for state storage
4. Creates/updates `.terraform.lock.hcl`
5. Creates `.terraform/` directory structure

---

## 2b. How Terraform Uses Providers

### Provider Architecture

```
Terraform Core
      │
      ├─ Plugin Discovery
      │
      ├─ RPC Communication
      │
      ▼
Provider Plugin (e.g., hashicorp/aws)
      │
      ├─ Resource Types (aws_instance, aws_vpc, ...)
      ├─ Data Source Types (aws_ami, aws_vpc, ...)
      └─ API Calls to Cloud Provider
```

**Terraform uses a plugin model:**
1. Terraform Core communicates with providers via RPC
2. Each provider is a separate binary (plugin)
3. Providers translate HCL resource definitions into API calls
4. Providers expose resource types and data sources

### Provider Configuration

```hcl
# Basic provider configuration
provider "aws" {
  region = "us-east-1"
}

# Provider with credentials (prefer environment variables or Vault!)
provider "aws" {
  region     = "us-east-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}

# Using environment variables (recommended)
# AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION
# No explicit provider block needed for default configuration
```

### Required vs Optional Provider Blocks

- A `provider` block without `required_providers` works but is **not recommended**
- Always declare providers in `required_providers` for explicit version management

```hcl
# ❌ Not recommended — implicit provider
provider "aws" {
  region = "us-east-1"
}

# ✅ Recommended — explicit provider declaration
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Provider Functions

Terraform 1.8+ supports provider-defined functions:

```hcl
# Example: using a provider function
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
}
```

---

## 2c. Multiple Provider Configurations

### Provider Aliases

Use aliases to configure multiple instances of the same provider (e.g., multiple AWS regions).

```hcl
provider "aws" {
  region = "us-east-1"  # Default provider
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

# Uses default provider (us-east-1)
resource "aws_instance" "east_app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Uses aliased provider
resource "aws_instance" "west_app" {
  provider      = aws.west
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "aws_instance" "eu_app" {
  provider      = aws.eu
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

### Multi-Cloud Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
}

# AWS resources
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Azure resources
resource "azurerm_resource_group" "main" {
  name     = "main-rg"
  location = "East US"
}
```

### Provider Aliases in Modules

```hcl
# Root module — defining providers
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Passing providers to modules
module "vpc_east" {
  source   = "./modules/vpc"
  providers = {
    aws = aws
  }
}

module "vpc_west" {
  source   = "./modules/vpc"
  providers = {
    aws = aws.west
  }
}
```

---

## 2d. How Terraform Uses and Manages State

### Purpose of Terraform State

Terraform state is the mechanism by which Terraform maps real-world infrastructure objects to your configuration. It is essential for Terraform to function.

| Purpose | Description |
|---------|-------------|
| **Resource Mapping** | Maps resource addresses (e.g., `aws_instance.web`) to real-world resource IDs (e.g., `i-1234567890abcdef0`) |
| **Metadata Storage** | Stores resource attributes, dependencies, and provider configuration |
| **Performance Caching** | Caches resource attributes so Terraform doesn't need to query APIs on every run |
| **Dependency Tracking** | Records the order resources were created and their dependencies |

### What State Contains

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
- `serial` — Incremented on each state update; used for locking/conflict detection
- `lineage` — Unique identifier for the state; prevents accidental state file swaps
- `outputs` — Output values from the configuration
- `resources` — All managed resources and their attributes

### Local State vs Remote State

| Aspect | Local State | Remote State |
|--------|-------------|--------------|
| **Storage** | `terraform.tfstate` on local disk | Backend (S3, HCP Terraform, etc.) |
| **Collaboration** | Not suitable for teams | Supports team collaboration |
| **Locking** | No built-in locking | Built-in locking (most backends) |
| **Security** | File on disk (risk of exposure) | Encrypted at rest |
| **Backup** | Manual or `.tfstate.backup` | Automatic versioning |
| **Use Case** | Individual/learning | Teams and production |

### Local State

```bash
# Default state file
terraform.tfstate

# Automatic backup after each apply
terraform.tfstate.backup
```

```hcl
# No backend block needed — local is default
# State is stored in terraform.tfstate in working directory
```

**Warning:** Local state files contain sensitive data. Never commit them to version control.

```bash
# Add to .gitignore
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl  # Optional — some teams DO commit this
```

### Remote State Overview

```hcl
# S3 backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# HCP Terraform backend
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}
```

**State locking** prevents concurrent operations that could corrupt state. Supported by most remote backends (S3+DynamoDB, HCP Terraform, Azure Blob, GCS, etc.).

### Why State Matters

Without state, Terraform would need to:
1. Query every cloud API on every run to discover existing resources
2. Have no way to know which resources it created vs. pre-existing ones
3. Re-create resources that already exist (breaking idempotency)

**State is not optional** — Terraform requires it to function correctly. Even without explicit backend configuration, local state is always used.

---

## Exam Tips

1. **`required_providers` is mandatory** — always declare providers with source and version
2. **`.terraform.lock.hcl` should be committed** — ensures reproducible builds
3. **Use `~>` for version constraints** — allows minor/patch updates, prevents major version jumps
4. **Provider aliases** — use the `alias` argument and `provider = aws.alias_name` in resources
5. **State maps config to reality** — without state, Terraform cannot manage resources
6. **State contains sensitive data** — always encrypt remote state and add `.tfstate` to `.gitignore`
7. **State locking** — always use a backend that supports locking for team environments
8. **`terraform init -upgrade`** — needed to update provider versions beyond the lock file

---

## Quick Review Questions

1. **Q:** What is the purpose of the `.terraform.lock.hcl` file?
   **A:** It records the exact provider versions used, ensuring consistent installations across team members and CI.

2. **Q:** What does the `~> 3.0` version constraint allow?
   **A:** Versions 3.x where x can be any value, but not 4.0 or higher.

3. **Q:** How do you configure multiple AWS regions in Terraform?
   **A:** Use provider aliases: `provider "aws" { alias = "west"; region = "us-west-2" }` and reference with `provider = aws.west`.

4. **Q:** Why is Terraform state necessary?
   **A:** It maps resource addresses to real-world IDs, caches attributes for performance, and tracks dependencies.

5. **Q:** Should you commit `.terraform.lock.hcl` to version control?
   **A:** Yes — it ensures all team members use the same provider versions.

6. **Q:** What are the main purposes of Terraform state?
   **A:** Resource mapping (config → real resources), metadata storage, performance caching, and dependency tracking.

---

[⬅️ Previous: IaC Concepts](01-iac-concepts.md) | [Next: Core Workflow ➡️](03-core-workflow.md)