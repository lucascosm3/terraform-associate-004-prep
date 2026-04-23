# 05 - Terraform Modules

> 📊 **Exam Weight:** 15%

---

## What are Modules?

Modules are containers for multiple resources that are used together. They enable code reuse, organization, and abstraction.

### Module Structure

```
my-module/
├── main.tf          # Core resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── README.md        # Documentation
└── versions.tf      # Provider requirements
```

### Root Module vs Child Modules

```
project/
├── main.tf              # Root module
├── variables.tf         # Root variables
├── outputs.tf           # Root outputs
└── modules/
    ├── networking/      # Child module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── compute/         # Child module
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## Creating Modules

### Module Files

**main.tf**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = var.vpc_name
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.vpc_name}-igw"
  }
}
```

**variables.tf**
```hcl
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
}

variable "enable_dns" {
  description = "Enable DNS support"
  type        = bool
  default     = true
}
```

**outputs.tf**
```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "igw_id" {
  description = "ID of the Internet Gateway"
  value       = aws_internet_gateway.main.id
}
```

**versions.tf**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
  }
}
```

---

## Using Modules

### Local Modules

```hcl
module "vpc" {
  source   = "./modules/vpc"
  vpc_cidr = "10.0.0.0/16"
  vpc_name = "production-vpc"
}

module "compute" {
  source        = "./modules/compute"
  instance_type = "t2.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]
}
```

### Terraform Registry Modules

```hcl
# AWS VPC Module from Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.2"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
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

### Module Sources

```hcl
# Local path
module "local" {
  source = "./modules/example"
}

# Terraform Registry
module "registry" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.0.0"
}

# GitHub
module "github" {
  source = "github.com/org/repo//modules/example?ref=v1.0.0"
}

# Git (generic)
module "git" {
  source = "git::https://example.com/repo.git//modules/example?ref=v1.0.0"
}

# S3 Bucket
module "s3" {
  source = "s3::https://s3.amazonaws.com/bucket/modules/example.zip"
}

# HTTP/HTTPS
module "http" {
  source = "https://example.com/modules/example.zip"
}
```

---

## Module Inputs & Outputs

### Passing Inputs

```hcl
module "database" {
  source = "./modules/rds"
  
  # Required variables
  db_name     = "production"
  db_username = "admin"
  db_password = var.db_password
  
  # Optional variables (using defaults)
  instance_class = "db.t3.medium"
  engine_version = "13.7"
}
```

### Accessing Outputs

```hcl
module "vpc" {
  source = "./modules/vpc"
  # ...
}

# Use module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_id
  
  tags = {
    vpc_id = module.vpc.vpc_id
  }
}
```

---

## Module Best Practices

### 1. Module Documentation

Always include a `README.md`:

```markdown
# VPC Module

Terraform module for creating a VPC with public and private subnets.

## Usage

```hcl
module "vpc" {
  source = "./modules/vpc"
  
  vpc_name = "production"
  vpc_cidr = "10.0.0.0/16"
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| vpc_name | Name of the VPC | string | n/a | yes |
| vpc_cidr | CIDR block | string | "10.0.0.0/16" | no |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | ID of the VPC |
```

### 2. Module Versioning

Always pin module versions:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.2"  # Pin to specific version
}

# Using version constraints
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.14"  # 3.14.x, but not 4.0.0
}
```

### 3. Module Composition

Build complex infrastructure by composing modules:

```hcl
# Main configuration
module "networking" {
  source = "./modules/vpc"
  # ...
}

module "database" {
  source = "./modules/rds"
  
  subnet_ids = module.networking.private_subnet_ids
  vpc_id     = module.networking.vpc_id
}

module "application" {
  source = "./modules/ecs"
  
  subnet_ids     = module.networking.public_subnet_ids
  database_url   = module.database.connection_string
}
```

### 4. Module Scope

Keep modules focused on a single responsibility:

```hcl
# ✅ Good: Specific purpose
module "vpc" { }
module "rds" { }
module "ec2_instance" { }

# ❌ Bad: Too many responsibilities
module "entire_infrastructure" { }
```

---

## Module Gotchas

### Module Instances are Isolated

Each module instance has its own isolated resources:

```hcl
module "vpc_prod" {
  source   = "./modules/vpc"
  vpc_name = "production"
}

module "vpc_dev" {
  source   = "./modules/vpc"
  vpc_name = "development"
}

# These create completely separate VPCs
```

### Circular Dependencies

Avoid circular dependencies between modules:

```hcl
# ❌ Bad: Circular dependency
module "a" {
  source = "./modules/a"
  input  = module.b.output
}

module "b" {
  source = "./modules/b"
  input  = module.a.output
}
```

### Sensitive Data

Mark sensitive outputs in modules:

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

---

## Module Registry

### Terraform Registry

- Official registry: [registry.terraform.io](https://registry.terraform.io)
- Verified modules (V) - maintained by HashiCorp partners
- Community modules

### Private Registry

- Terraform Cloud/Enterprise private registry
- Self-hosted module registries

```hcl
# Private Terraform Cloud registry
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.0.0"
}
```

---

## Exam Tips

1. **Module Structure** - Know the standard files: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`
2. **Module Sources** - Local, Registry, Git, S3, HTTP
3. **Version Pinning** - Always use `version` parameter for registry modules
4. **Module Outputs** - Access via `module.<name>.<output>`
5. **Root vs Child** - Root is main config; child is called module
6. **Composition** - Build complex systems by composing simple modules

---

[⬅️ Previous: CLI Commands](04-cli-commands.md) | [Next: Workflow ➡️](06-workflow.md)
