# Lab 05: Terraform Workspaces

## 🎯 Objective

Learn to use Terraform workspaces to manage multiple environments with the same configuration.

---

## 📋 Prerequisites

- AWS account with credentials
- Terraform installed
- Understanding of variables

---

## 📝 Steps

### Step 1: Create Base Configuration

```bash
mkdir lab-05-workspaces
cd lab-05-workspaces
```

**main.tf**
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
  region = lookup(var.region, terraform.workspace, "us-west-2")
}

# Local values based on workspace
locals {
  workspace_config = lookup(var.workspace_configs, terraform.workspace, var.workspace_configs["default"])
  
  common_tags = {
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
    Project     = "workspace-lab"
  }
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

# VPC
resource "aws_vpc" "main" {
  cidr_block           = local.workspace_config.vpc_cidr
  enable_dns_hostnames = true
  
  tags = merge(local.common_tags, {
    Name = "${terraform.workspace}-vpc"
  })
}

# Subnet
resource "aws_subnet" "main" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(local.workspace_config.vpc_cidr, 8, 1)
  availability_zone       = "${lookup(var.region, terraform.workspace, "us-west-2")}a"
  map_public_ip_on_launch = true
  
  tags = merge(local.common_tags, {
    Name = "${terraform.workspace}-subnet"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(local.common_tags, {
    Name = "${terraform.workspace}-igw"
  })
}

# Route table
resource "aws_route_table" "main" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(local.common_tags, {
    Name = "${terraform.workspace}-rt"
  })
}

# Route table association
resource "aws_route_table_association" "main" {
  subnet_id      = aws_subnet.main.id
  route_table_id = aws_route_table.main.id
}

# Security Group
resource "aws_security_group" "web" {
  name_prefix = "${terraform.workspace}-web"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = local.workspace_config.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = local.common_tags
}

# EC2 Instances
resource "aws_instance" "web" {
  count = local.workspace_config.instance_count
  
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = local.workspace_config.instance_type
  subnet_id              = aws_subnet.main.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from ${terraform.workspace} - Instance ${count.index + 1}</h1>" > /var/www/html/index.html
              EOF
  
  tags = merge(local.common_tags, {
    Name = "${terraform.workspace}-web-${count.index + 1}"
  })
}
```

**variables.tf**
```hcl
variable "region" {
  description = "Region per workspace"
  type        = map(string)
  default = {
    default = "us-west-2"
    dev     = "us-west-2"
    staging = "us-east-1"
    prod    = "us-east-1"
  }
}

variable "workspace_configs" {
  description = "Configuration per workspace"
  type = map(object({
    vpc_cidr       = string
    instance_type  = string
    instance_count = number
    allowed_ports  = list(number)
  }))
  
  default = {
    default = {
      vpc_cidr       = "10.0.0.0/16"
      instance_type  = "t2.micro"
      instance_count = 1
      allowed_ports  = [80, 22]
    }
    dev = {
      vpc_cidr       = "10.1.0.0/16"
      instance_type  = "t2.micro"
      instance_count = 1
      allowed_ports  = [80, 443, 22, 8080]
    }
    staging = {
      vpc_cidr       = "10.2.0.0/16"
      instance_type  = "t2.small"
      instance_count = 2
      allowed_ports  = [80, 443, 22]
    }
    prod = {
      vpc_cidr       = "10.3.0.0/16"
      instance_type  = "t3.medium"
      instance_count = 3
      allowed_ports  = [80, 443]
    }
  }
}
```

**outputs.tf**
```hcl
output "workspace_name" {
  value = terraform.workspace
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "instance_count" {
  value = length(aws_instance.web)
}

output "instance_ips" {
  value = aws_instance.web[*].public_ip
}

output "environment_config" {
  value = {
    region         = lookup(var.region, terraform.workspace, "us-west-2")
    vpc_cidr       = local.workspace_config.vpc_cidr
    instance_type  = local.workspace_config.instance_type
    instance_count = local.workspace_config.instance_count
  }
}
```

### Step 2: Initialize Terraform

```bash
terraform init
```

### Step 3: List and Show Current Workspace

```bash
# List all workspaces
terraform workspace list

# Show current workspace
terraform workspace show
```

**Output:**
```
* default
```

### Step 4: Create Dev Workspace

```bash
# Create new workspace
terraform workspace new dev

# Verify
terraform workspace show
# Output: dev
```

### Step 5: Apply Dev Workspace

```bash
terraform plan
terraform apply
```

**Observe:**
- Configuration uses `dev` workspace values
- Different CIDR block (10.1.0.0/16)
- 1 instance of t2.micro
- More ports allowed

### Step 6: Create Staging Workspace

```bash
terraform workspace new staging
terraform workspace list
terraform apply
```

### Step 7: Create Prod Workspace

```bash
terraform workspace new prod
terraform apply
```

### Step 8: Switch Between Workspaces

```bash
# Switch to dev
terraform workspace select dev
terraform output

# Switch to prod
terraform workspace select prod
terraform output

# Compare
terraform workspace select dev
echo "Dev instances: $(terraform output -raw instance_count)"

terraform workspace select prod
echo "Prod instances: $(terraform output -raw instance_count)"
```

### Step 9: View State Files

Workspaces store state separately:

```bash
# Local state structure
ls -la terraform.tfstate.d/

# List workspace state directories
tree terraform.tfstate.d/

# Each workspace has its own state
ls terraform.tfstate.d/dev/
ls terraform.tfstate.d/prod/
```

### Step 10: Workspace Best Practices Demo

**backend.tf** (Optional - for remote state)
```hcl
terraform {
  backend "s3" {
    bucket = "your-state-bucket"
    key    = "workspaces/terraform.tfstate"
    region = "us-west-2"
    
    workspace_key_prefix = "workspaces"
    # This creates:
    # - workspaces/dev/terraform.tfstate
    # - workspaces/prod/terraform.tfstate
  }
}
```

---

## 🎓 What You Learned

1. **Workspace Commands** - list, show, new, select, delete
2. **Workspace Variables** - Use `terraform.workspace` value
3. **State Separation** - Each workspace has isolated state
4. **Configuration per Workspace** - Different settings per environment
5. **Remote State with Workspaces** - Automatic key prefixing

---

## 📝 Key Concepts

### Workspace Commands

```bash
terraform workspace list          # List workspaces
terraform workspace show          # Show current
terraform workspace new <name>    # Create workspace
terraform workspace select <name> # Switch workspace
terraform workspace delete <name> # Delete workspace
```

### Using terraform.workspace

```hcl
# In configuration
resource "aws_instance" "web" {
  tags = {
    Environment = terraform.workspace
  }
}

# In locals with lookup
locals {
  config = lookup(var.configs, terraform.workspace, var.configs["default"])
}
```

### Workspace vs Directory Structure

**Workspaces:**
- Single codebase
- Separate state per workspace
- Good for: environments with similar configs

**Directory Structure:**
- Different codebases
- Complete separation
- Good for: significantly different environments

---

## 🚀 Challenge Exercises

1. **Configure S3 backend** with workspace_key_prefix
2. **Add a test workspace** with minimal resources
3. **Create workspace-specific variables** using .tfvars files
4. **Use conditionals** based on workspace (e.g., prod gets more monitoring)

---

## 🧹 Cleanup

```bash
# Destroy each workspace
terraform workspace select dev
terraform destroy -auto-approve

terraform workspace select staging
terraform destroy -auto-approve

terraform workspace select prod
terraform destroy -auto-approve

# Delete workspaces (must switch away first)
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```

---

[⬅️ Previous Lab](../lab-04-modules-basics/) | [Back to Labs ➡️](../)
