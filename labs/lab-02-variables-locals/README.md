# Lab 02: Variables and Locals

## 🎯 Objective

Learn to use variables and locals to make your Terraform configurations flexible and reusable.

---

## 📋 Prerequisites

- AWS account with credentials configured
- Terraform installed
- Completion of Lab 01

---

## 📝 Steps

### Step 1: Create Project Directory

```bash
mkdir lab-02-variables-locals
cd lab-02-variables-locals
```

### Step 2: Create Configuration Files

**variables.tf**
```hcl
# Required variable (no default)
variable "project_name" {
  description = "Name of the project"
  type        = string
}

# Variable with default
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

# Variable with validation
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium", "t3.micro"], var.instance_type)
    error_message = "Instance type must be one of: t2.micro, t2.small, t2.medium, t3.micro"
  }
}

# Number variable
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 5
    error_message = "Instance count must be between 1 and 5."
  }
}

# Boolean variable
variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

# List variable
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b"]
}

# Map variable
variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    ManagedBy = "Terraform"
    Team      = "DevOps"
  }
}

# Complex object variable
variable "vpc_config" {
  description = "VPC configuration"
  type = object({
    cidr_block = string
    subnets = list(object({
      name = string
      cidr = string
      az   = string
    }))
  })
  default = {
    cidr_block = "10.0.0.0/16"
    subnets = [
      {
        name = "public-1"
        cidr = "10.0.1.0/24"
        az   = "us-west-2a"
      },
      {
        name = "public-2"
        cidr = "10.0.2.0/24"
        az   = "us-west-2b"
      }
    ]
  }
}

# Variable with sensitive data
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
  default     = "change-me-in-production"
}
```

**main.tf**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

# Locals for computed values
locals {
  # Merge common tags with specific ones
  default_tags = merge(
    var.common_tags,
    {
      Project     = var.project_name
      Environment = var.environment
    }
  )
  
  # Compute a name prefix
  name_prefix = "${var.project_name}-${var.environment}"
  
  # Conditional logic
  monitoring_enabled = var.enable_monitoring ? "enabled" : "disabled"
}

# Data source for AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# VPC using object variable
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_config.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(local.default_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# Subnets using dynamic blocks
resource "aws_subnet" "main" {
  count = length(var.vpc_config.subnets)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.vpc_config.subnets[count.index].cidr
  availability_zone       = var.vpc_config.subnets[count.index].az
  map_public_ip_on_launch = true
  
  tags = merge(local.default_tags, {
    Name = "${local.name_prefix}-${var.vpc_config.subnets[count.index].name}"
  })
}

# Multiple instances using count
resource "aws_instance" "web" {
  count = var.instance_count
  
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.main[count.index % length(aws_subnet.main)].id
  vpc_security_group_ids = [aws_security_group.main.id]
  
  monitoring = var.enable_monitoring
  
  tags = merge(local.default_tags, {
    Name = "${local.name_prefix}-web-${count.index + 1}"
  })
}

# Security group
resource "aws_security_group" "main" {
  name_prefix = "${local.name_prefix}-sg"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = local.default_tags
  
  lifecycle {
    create_before_destroy = true
  }
}
```

**outputs.tf**
```hcl
output "instance_ids" {
  description = "IDs of created instances"
  value       = aws_instance.web[*].id
}

output "instance_public_ips" {
  description = "Public IPs of instances"
  value       = aws_instance.web[*].public_ip
}

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "Subnet IDs"
  value       = aws_subnet.main[*].id
}

# Sensitive output
output "database_password" {
  description = "Database password"
  value       = var.db_password
  sensitive   = true
}

# Computed output
output "instance_summary" {
  description = "Summary of instances"
  value = {
    count             = length(aws_instance.web)
    monitoring_status = local.monitoring_enabled
  }
}
```

### Step 3: Create Terraform.tfvars

**terraform.tfvars**
```hcl
project_name = "myapp"
environment  = "staging"

instance_type  = "t2.small"
instance_count = 2

enable_monitoring = true

common_tags = {
  ManagedBy = "Terraform"
  Team      = "Platform"
  CostCenter = "12345"
}

db_password = "SecureP@ssw0rd123!"
```

### Step 4: Initialize and Plan

```bash
terraform init
terraform plan
```

**Observe:**
- How variables are substituted
- How locals are computed
- How validation works (try invalid values)

### Step 5: Apply

```bash
terraform apply
```

### Step 6: Test Variable Overrides

```bash
# Override specific variable
terraform plan -var="instance_count=3" -var="environment=production"

# Use different var file
terraform plan -var-file="production.tfvars"
```

### Step 7: View Sensitive Output

```bash
# This will mask the password
terraform output

# This will show the actual value
terraform output -raw database_password
```

---

## 🎓 What You Learned

1. **Variable Types** - string, number, bool, list, map, object, tuple
2. **Variable Validation** - Enforce constraints on inputs
3. **Sensitive Variables** - Protect sensitive data in outputs
4. **Locals** - Computed values and reusable expressions
5. **Variable Files** - `.tfvars` for environment-specific values
6. **Variable Precedence** - Command line > .tfvars > defaults

---

## 📝 Key Concepts

### Variable Precedence (Highest to Lowest)

1. Command-line flags: `-var="key=value"`
2. Variable files: `-var-file="file.tfvars"`
3. `terraform.tfvars` (auto-loaded)
4. `*.auto.tfvars` (auto-loaded, alphabetical order)
5. Environment variables: `TF_VAR_name`
6. Default values

### Variable Types

```hcl
string    # "hello"
number    # 42
bool      # true/false
list      # ["a", "b", "c"]
map       # {key = "value"}
object    # Complex structure with typed attributes
tuple     # Fixed-length list with typed elements
```

### Locals vs Variables

| | Variables | Locals |
|--|-----------|--------|
| Input | From user | Computed |
| Override | Yes | No |
| Purpose | Configuration | Reuse expressions |

---

## 🚀 Challenge Exercises

1. **Add more validation rules** to variables
2. **Create environment-specific var files** (dev.tfvars, prod.tfvars)
3. **Use `for_each` instead of `count`** for instances
4. **Create a variable for allowed CIDR blocks** with validation

---

## 🧹 Cleanup

```bash
terraform destroy -auto-approve
```

---

[⬅️ Previous Lab](../lab-01-first-configuration/) | [Next Lab: State Remote ➡️](../lab-03-state-remote/)
