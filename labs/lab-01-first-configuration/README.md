# Lab 01: First Terraform Configuration

## 🎯 Objective

Create your first Terraform configuration to provision an AWS EC2 instance and understand the basic workflow.

---

## 📋 Prerequisites

- AWS account (Free Tier works fine)
- Terraform installed (v1.0+)
- AWS CLI configured with credentials

---

## 📝 Steps

### Step 1: Create Project Directory

```bash
mkdir lab-01-first-configuration
cd lab-01-first-configuration
```

### Step 2: Create Configuration Files

**main.tf**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  required_version = ">= 1.2.0"
}

provider "aws" {
  region = "us-west-2"
}

# Get latest Amazon Linux 2 AMI
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

# Create a VPC
resource "aws_vpc" "lab" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "lab-vpc"
  }
}

# Create a public subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.lab.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-west-2a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "lab-public-subnet"
  }
}

# Create Internet Gateway
resource "aws_internet_gateway" "lab" {
  vpc_id = aws_vpc.lab.id
  
  tags = {
    Name = "lab-igw"
  }
}

# Create route table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.lab.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.lab.id
  }
  
  tags = {
    Name = "lab-public-rt"
  }
}

# Associate route table with subnet
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Create security group
resource "aws_security_group" "lab" {
  name_prefix = "lab-sg"
  vpc_id      = aws_vpc.lab.id
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "SSH access"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "lab-sg"
  }
}

# Create EC2 instance
resource "aws_instance" "lab" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.lab.id]
  
  tags = {
    Name = "lab-instance"
  }
}
```

**outputs.tf**
```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.lab.id
}

output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.lab.public_ip
}

output "instance_private_ip" {
  description = "Private IP address of the EC2 instance"
  value       = aws_instance.lab.private_ip
}

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.lab.id
}
```

### Step 3: Initialize Terraform

```bash
terraform init
```

**Expected output:**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
- Installed hashicorp/aws v5.x.x (signed by HashiCorp)

Terraform has been successfully initialized!
```

### Step 4: Format and Validate

```bash
terraform fmt
terraform validate
```

### Step 5: Create Execution Plan

```bash
terraform plan
```

**Review the output and observe:**
- How many resources will be created
- Resource dependencies
- Resource attributes

### Step 6: Apply Configuration

```bash
terraform apply
```

Type `yes` when prompted.

### Step 7: Verify Outputs

```bash
terraform output
```

### Step 8: Inspect State

```bash
terraform state list
terraform state show aws_instance.lab
terraform show
```

### Step 9: Destroy Infrastructure

```bash
terraform destroy
```

Type `yes` when prompted.

---

## 🎓 What You Learned

1. **HCL Syntax** - Blocks, arguments, and resource definitions
2. **Data Sources** - Query existing data (AMI lookup)
3. **Resource Dependencies** - Implicit dependencies (vpc → subnet → instance)
4. **Workflow** - init → validate → plan → apply → destroy
5. **State Management** - Terraform tracks resources in state
6. **Outputs** - Expose useful information

---

## 📝 Key Concepts Review

### Implicit Dependencies
```hcl
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.lab.id  # Terraform knows to create VPC first
}
```

### Data Sources
```hcl
data "aws_ami" "amazon_linux" {
  # Queries AWS API to find AMI
}
```

### State File
- Created after first `apply`
- Maps resource addresses to real-world IDs
- Do NOT commit to Git!

---

## 🚀 Challenge Exercises

1. **Add a second subnet** in a different availability zone
2. **Add an Elastic IP** and associate it with the instance
3. **Add more ingress rules** (HTTP on port 80)
4. **Use variables** for the region and instance type

---

## 🧹 Cleanup

```bash
terraform destroy -auto-approve
rm -rf .terraform/
rm -f terraform.tfstate*
```

---

[⬅️ Back to Labs](../) | [Next Lab: Variables & Locals ➡️](../lab-02-variables-locals/)
