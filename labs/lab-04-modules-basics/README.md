# Lab 04: Terraform Modules

## 🎯 Objective

Create and use Terraform modules to build reusable, composable infrastructure components.

---

## 📋 Prerequisites

- AWS account with credentials
- Terraform installed
- Understanding of variables and outputs

---

## 📝 Steps

### Step 1: Create Module Structure

```bash
mkdir -p lab-04-modules-basics/modules/{vpc,compute,database}
mkdir -p lab-04-modules-basics/environments/{dev,prod}
cd lab-04-modules-basics
```

### Step 2: Create VPC Module

**modules/vpc/main.tf**
```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support
  
  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}

resource "aws_internet_gateway" "this" {
  count = var.create_igw ? 1 : 0
  
  vpc_id = aws_vpc.this.id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-igw"
    }
  )
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)
  
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnets[count.index].cidr
  availability_zone       = var.public_subnets[count.index].az
  map_public_ip_on_launch = true
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-public-${count.index + 1}"
      Type = "Public"
    }
  )
}

resource "aws_subnet" "private" {
  count = length(var.private_subnets)
  
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index].cidr
  availability_zone = var.private_subnets[count.index].az
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-private-${count.index + 1}"
      Type = "Private"
    }
  )
}
```

**modules/vpc/variables.tf**
```hcl
variable "name" {
  description = "Name of the VPC"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
}

variable "enable_dns_hostnames" {
  description = "Enable DNS hostnames"
  type        = bool
  default     = true
}

variable "enable_dns_support" {
  description = "Enable DNS support"
  type        = bool
  default     = true
}

variable "create_igw" {
  description = "Create Internet Gateway"
  type        = bool
  default     = true
}

variable "public_subnets" {
  description = "List of public subnet configurations"
  type = list(object({
    cidr = string
    az   = string
  }))
  default = []
}

variable "private_subnets" {
  description = "List of private subnet configurations"
  type = list(object({
    cidr = string
    az   = string
  }))
  default = []
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

**modules/vpc/outputs.tf**
```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "ID of Internet Gateway"
  value       = var.create_igw ? aws_internet_gateway.this[0].id : null
}
```

**modules/vpc/versions.tf**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

### Step 3: Create Compute Module

**modules/compute/main.tf**
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "this" {
  count = var.instance_count
  
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids
  key_name               = var.key_name
  
  user_data = var.user_data
  
  monitoring = var.monitoring
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-${count.index + 1}"
    }
  )
}
```

**modules/compute/variables.tf**
```hcl
variable "name" {
  description = "Name prefix for instances"
  type        = string
}

variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_id" {
  description = "Subnet ID for instances"
  type        = string
}

variable "security_group_ids" {
  description = "List of security group IDs"
  type        = list(string)
  default     = []
}

variable "key_name" {
  description = "SSH key pair name"
  type        = string
  default     = null
}

variable "user_data" {
  description = "User data script"
  type        = string
  default     = null
}

variable "monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags to apply"
  type        = map(string)
  default     = {}
}
```

**modules/compute/outputs.tf**
```hcl
output "instance_ids" {
  description = "IDs of created instances"
  value       = aws_instance.this[*].id
}

output "instance_public_ips" {
  description = "Public IP addresses"
  value       = aws_instance.this[*].public_ip
}

output "instance_private_ips" {
  description = "Private IP addresses"
  value       = aws_instance.this[*].private_ip
}
```

### Step 4: Create Dev Environment

**environments/dev/main.tf**
```hcl
terraform {
  required_version = ">= 1.0"
  
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

locals {
  environment = "dev"
  common_tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
  }
}

# VPC Module
module "vpc" {
  source = "../../modules/vpc"
  
  name       = "myapp-${local.environment}"
  cidr_block = "10.0.0.0/16"
  
  public_subnets = [
    { cidr = "10.0.1.0/24", az = "us-west-2a" },
    { cidr = "10.0.2.0/24", az = "us-west-2b" }
  ]
  
  private_subnets = [
    { cidr = "10.0.3.0/24", az = "us-west-2a" },
    { cidr = "10.0.4.0/24", az = "us-west-2b" }
  ]
  
  tags = local.common_tags
}

# Security Group
resource "aws_security_group" "web" {
  name_prefix = "myapp-${local.environment}-web"
  vpc_id      = module.vpc.vpc_id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = local.common_tags
}

# Compute Module
module "web" {
  source = "../../modules/compute"
  
  name             = "myapp-${local.environment}-web"
  instance_count   = 2
  instance_type    = "t2.micro"
  subnet_id        = module.vpc.public_subnet_ids[0]
  security_group_ids = [aws_security_group.web.id]
  
  user_data = templatefile("${path.module}/user_data.sh", {
    environment = local.environment
  })
  
  tags = local.common_tags
}
```

**environments/dev/user_data.sh**
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from ${environment}!</h1>" > /var/www/html/index.html
```

**environments/dev/outputs.tf**
```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnet_ids" {
  value = module.vpc.public_subnet_ids
}

output "web_instance_ips" {
  value = module.web.instance_public_ips
}
```

### Step 5: Apply Dev Environment

```bash
cd environments/dev
terraform init
terraform plan
terraform apply
```

### Step 6: Create Prod Environment

**environments/prod/main.tf**
```hcl
terraform {
  required_version = ">= 1.0"
  
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

locals {
  environment = "prod"
  common_tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
  }
}

module "vpc" {
  source = "../../modules/vpc"
  
  name       = "myapp-${local.environment}"
  cidr_block = "172.16.0.0/16"
  
  public_subnets = [
    { cidr = "172.16.1.0/24", az = "us-west-2a" },
    { cidr = "172.16.2.0/24", az = "us-west-2b" },
    { cidr = "172.16.3.0/24", az = "us-west-2c" }
  ]
  
  private_subnets = [
    { cidr = "172.16.4.0/24", az = "us-west-2a" },
    { cidr = "172.16.5.0/24", az = "us-west-2b" },
    { cidr = "172.16.6.0/24", az = "us-west-2c" }
  ]
  
  tags = local.common_tags
}

# Security group with more restrictive rules
resource "aws_security_group" "web" {
  name_prefix = "myapp-${local.environment}-web"
  vpc_id      = module.vpc.vpc_id
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # More restrictive
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = local.common_tags
}

module "web" {
  source = "../../modules/compute"
  
  name             = "myapp-${local.environment}-web"
  instance_count   = 3
  instance_type    = "t3.medium"
  subnet_id        = module.vpc.public_subnet_ids[0]
  security_group_ids = [aws_security_group.web.id]
  monitoring       = true
  
  tags = local.common_tags
}
```

### Step 7: Apply Prod Environment

```bash
cd ../prod
terraform init
terraform plan
terraform apply
```

---

## 🎓 What You Learned

1. **Module Structure** - main.tf, variables.tf, outputs.tf, versions.tf
2. **Module Inputs** - Variables as module API
3. **Module Outputs** - Expose important information
4. **Module Composition** - Combine modules for complex infrastructure
5. **Environment Separation** - Different configs per environment
6. **Module Versioning** - Pin module versions for stability

---

## 📝 Key Concepts

### Module Registry Usage

```hcl
# Public registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  # ...
}
```

### Module Sources

```hcl
# Local path
source = "./modules/vpc"

# GitHub
source = "github.com/org/repo//modules/vpc?ref=v1.0.0"

# Terraform Registry
source = "terraform-aws-modules/vpc/aws"

# S3
source = "s3::https://s3.amazonaws.com/bucket/modules/vpc.zip"
```

---

## 🚀 Challenge Exercises

1. **Create a database module** for RDS
2. **Add validation** to module variables
3. **Publish module** to Terraform Cloud private registry
4. **Create a composite module** that uses vpc + compute

---

## 🧹 Cleanup

```bash
# Destroy both environments
cd environments/dev
terraform destroy -auto-approve

cd ../prod
terraform destroy -auto-approve
```

---

[⬅️ Previous Lab](../lab-03-state-remote/) | [Next Lab: Workspaces ➡️](../lab-05-workspaces/)
