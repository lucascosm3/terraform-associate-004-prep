# 05 - Domain 5: Terraform Modules

> Exam Weight: ~11% | Tested on Terraform 1.12

---

## Objectives

- [x] **5a:** Explain how Terraform sources modules
- [x] **5b:** Describe variable scope within modules
- [x] **5c:** Use modules in configuration
- [x] **5d:** Manage module versions

---

## 5a. How Terraform Sources Modules

### What Are Modules?

Modules are containers for multiple resources that are used together. Every Terraform configuration has at least one module — the **root module** (the working directory). Modules called from the root module are **child modules**.

### Module Structure

```
my-module/
├── main.tf          # Core resources
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── versions.tf      # Provider and Terraform version requirements
└── README.md        # Module documentation
```

### Module Source Types

```hcl
# Local path (relative)
module "vpc" {
  source = "./modules/vpc"
}

# Local path (absolute)
module "vpc" {
  source = "/home/user/terraform/modules/vpc"
}

# Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}

# GitHub
module "vpc" {
  source = "github.com/org/terraform-modules//vpc?ref=v1.0.0"
}

# Git (generic)
module "vpc" {
  source = "git::https://example.com/repo.git//modules/vpc?ref=v1.0.0"
}

# S3 Bucket
module "vpc" {
  source = "s3::https://s3.amazonaws.com/bucket/modules/vpc.zip"
}

# GCS Bucket
module "vpc" {
  source = "gcs::https://www.googleapis.com/storage/v1/bucket/modules/vpc.zip"
}

# HTTP/HTTPS
module "vpc" {
  source = "https://example.com/modules/vpc.zip"
}
```

### Module Source Comparison

| Source Type | Version Constraints | Use Case |
|-------------|---------------------|----------|
| Local path | No | Development, simple projects |
| Registry | Yes | Community/verified modules |
| GitHub | Yes (via `?ref=`) | Private org modules |
| Git | Yes (via `?ref=`) | Private Git repositories |
| S3 | No | Private distribution |
| HTTP | No | Third-party archives |

### When to Run `terraform init`

You must run `terraform init` after:
- Adding a new `module` block
- Changing the `source` of an existing module
- Changing the `version` constraint of a module

```bash
# After adding or changing modules
terraform init

# Update modules to latest matching version
terraform init -upgrade

# Download modules only
terraform get
terraform get -update  # Update to latest matching versions
```

---

## 5b. Variable Scope Within Modules

### Scope Rules

1. **Variables declared in a module are only accessible within that module**
2. **Child modules receive values via the `module` block arguments**
3. **Parent modules access child module outputs via `module.<name>.<output>`**

```
Root Module
│
├── variable "vpc_cidr"    ← Root-level variable
│
├── module "vpc"            ← Child module call
│   ├── vpc_cidr = var.vpc_cidr     ← Passing root variable to child
│   └── environment = "production"   ← Passing literal value to child
│
├── module "app"            ← Another child module
│   └── subnet_ids = module.vpc.subnet_ids  ← Using output from another module
│
└── output "vpc_id"        ← Root-level output (from module output)
    └── value = module.vpc.vpc_id
```

### Module Input Variables

```hcl
# modules/vpc/variables.tf
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "public_subnets" {
  description = "List of public subnet CIDR blocks"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}
```

### Module Outputs

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "List of subnet IDs"
  value       = aws_subnet.public[*].id
}
```

### Accessing Module Outputs in Parent

```hcl
module "vpc" {
  source      = "./modules/vpc"
  vpc_cidr    = "10.0.0.0/16"
  environment = "production"
}

module "app" {
  source      = "./modules/app"
  subnet_ids  = module.vpc.subnet_ids
  vpc_id      = module.vpc.vpc_id
}
```

### Module Variable Scope Rules

| Concept | Description |
|---------|-------------|
| **Inputs** | Passed to the module via the `module` block |
| **Outputs** | Exposed by the module, accessed as `module.<name>.<output>` |
| **Locals** | Only visible within the module where they're defined |
| **Variables** | Only visible within the module where they're declared |
| **Providers** | Inherited from the root module unless explicitly passed |

### Passing Providers to Modules

```hcl
# Root module — defining provider aliases
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Passing a specific provider to a module
module "vpc_west" {
  source    = "./modules/vpc"
  providers = {
    aws = aws.west
  }
  vpc_cidr = "10.0.0.0/16"
}
```

---

## 5c. Using Modules in Configuration

### Local Modules

```hcl
# Root module: main.tf
module "vpc" {
  source   = "./modules/vpc"
  vpc_cidr = "10.0.0.0/16"
  environment = "production"
}

module "database" {
  source      = "./modules/rds"
  subnet_ids  = module.vpc.subnet_ids
  vpc_id      = module.vpc.vpc_id
  db_name     = "production"
  db_username = "admin"
  db_password = var.db_password
}

module "app" {
  source         = "./modules/ecs"
  subnet_ids     = module.vpc.public_subnet_ids
  db_connection  = module.database.connection_string
}
```

### Terraform Registry Modules

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}
```

**Registry module source format:** `<NAMESPACE>/<NAME>/<PROVIDER>`

### Module Composition Pattern

```hcl
# Build complex infrastructure by composing modules
module "networking" {
  source      = "./modules/vpc"
  vpc_cidr    = var.vpc_cidr
  environment = var.environment
}

module "database" {
  source     = "./modules/rds"
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
}

module "application" {
  source        = "./modules/ecs"
  vpc_id        = module.networking.vpc_id
  subnet_ids    = module.networking.public_subnet_ids
  database_url  = module.database.connection_string
}
```

### Module Best Practices

1. **Single Responsibility** — Each module should manage one logical component
2. **Document with README** — Include usage examples, inputs, and outputs
3. **Pin versions** — Always specify `version` for registry modules
4. **Use variables.tf and outputs.tf** — Keep declarations in separate files
5. **Validate inputs** — Use type constraints and validation blocks
6. **Mark sensitive outputs** — Use `sensitive = true` for secrets

```hcl
# Good module structure
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── README.md
├── rds/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── README.md
└── ecs/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    └── README.md
```

---

## 5d. Module Versions

### Version Constraints for Registry Modules

```hcl
# Pin to exact version (most restrictive)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}

# Allow patch updates (~> 5.1 → 5.1.x, not 5.2.0)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"
}

# Allow minor updates (~> 5 → 5.x, not 6.0)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5"
}

# Range constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0, < 6.0"
}
```

### Version Constraint Operators

| Operator | Meaning | Example | Allowed |
|----------|---------|---------|---------|
| `=` | Exact version | `= 5.1.2` | 5.1.2 only |
| `!=` | Not equal | `!= 5.1.2` | Any version except 5.1.2 |
| `>` | Greater than | `> 5.0` | 5.1, 6.0, etc. |
| `>=` | Greater than or equal | `>= 5.0` | 5.0, 5.1, 6.0, etc. |
| `<` | Less than | `< 6.0` | 5.x, 4.x, etc. |
| `<=` | Less than or equal | `<= 5.5` | 5.5, 5.4, etc. |
| `~>` | Allows rightmost version to increment | `~> 5.1` | 5.1, 5.2, etc. (not 6.0) |
| `~>` | Allows patch updates only | `~> 5.1.2` | 5.1.2, 5.1.3, etc. (not 5.2.0) |

### Module Versioning with Git

```hcl
# Using a Git tag as version
module "vpc" {
  source = "git::https://github.com/org/terraform-modules.git//vpc?ref=v2.1.0"
}

# Using a Git branch
module "vpc" {
  source = "git::https://github.com/org/terraform-modules.git//vpc?ref=main"
}

# Using a specific commit
module "vpc" {
  source = "git::https://github.com/org/terraform-modules.git//vpc?ref=abc123def456"
}
```

### Updating Module Versions

```bash
# Download modules for the first time
terraform init

# Update to latest versions matching constraints
terraform init -upgrade

# Download/update modules only
terraform get -update
```

**Important:** After changing a module source or version, you MUST run `terraform init` to download the new version.

### Private Registry (HCP Terraform)

```hcl
# HCP Terraform Private Registry
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.0.0"
}
```

**Benefits of Private Registry:**
- Share modules within your organization
- Version control for internal modules
- Access control and permissions
- Integration with Sentinel policies

---

## Module Gotchas

### Module Instances Are Isolated

```hcl
module "vpc_prod" {
  source   = "./modules/vpc"
  vpc_name = "production"
  vpc_cidr = "10.0.0.0/16"
}

module "vpc_dev" {
  source   = "./modules/vpc"
  vpc_name = "development"
  vpc_cidr = "172.16.0.0/16"
}
# These create completely separate VPCs with isolated state entries
```

### Circular Dependencies

```hcl
# ❌ BAD: Circular dependency between modules
module "a" {
  source = "./modules/a"
  input  = module.b.output
}

module "b" {
  source = "./modules/b"
  input  = module.a.output
}
```

**Solution:** Refactor to remove the circular dependency, or use data sources to read needed information.

### Module Count and for_each

```hcl
# Modules support count and for_each
module "vpc" {
  count   = length(var.environments)
  source  = "./modules/vpc"

  environment = var.environments[count.index]
  vpc_cidr    = var.vpc_cidrs[count.index]
}

# Using for_each with modules
module "vpc" {
  for_each = var.environments
  source   = "./modules/vpc"

  environment = each.key
  vpc_cidr    = each.value
}
```

---

## Exam Tips

1. **Module source types** — Know all source types: local, registry, Git, GitHub, S3, HTTP
2. **Run `terraform init`** — Required after adding/changing module sources
3. **Variable scope** — Variables are scoped to their module; pass values via the module block
4. **Accessing outputs** — Use `module.<name>.<output>` to access child module outputs
5. **Pin versions** — Always use `version` for registry modules
6. **`~>` operator** — Know how it allows patch vs minor updates
7. **Module composition** — Chain modules by connecting outputs to inputs
8. **Single responsibility** — Each module should manage one logical component
9. **No circular dependencies** — Modules cannot have circular references
10. **Private Registry** — HCP Terraform offers a private module registry

---

## Quick Review Questions

1. **Q:** What are the standard files in a Terraform module?
   **A:** `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, and `README.md`.

2. **Q:** How do you reference an output from a child module?
   **A:** `module.<module_name>.<output_name>`, e.g., `module.vpc.vpc_id`.

3. **Q:** What happens if you change a module's `source` or `version`?
   **A:** You must run `terraform init` to download the new version.

4. **Q:** What does `version = "~> 3.14"` mean?
   **A:** Allow version 3.14 and any 3.x version above that, but not 4.0 or higher (actually, `~> 3.14` allows 3.14.x and 3.15, etc. but NOT 4.0 — specifically, it allows >= 3.14.0 and < 4.0.0).

5. **Q:** Can modules use `count` and `for_each`?
   **A:** Yes, modules support both `count` and `for_each` meta-arguments.

6. **Q:** How do you pass a specific provider to a module?
   **A:** Use the `providers` map: `providers = { aws = aws.west }`.

---

[⬅️ Previous: Terraform Configuration](04-terraform-configuration.md) | [Next: State Management ➡️](06-state-management.md)