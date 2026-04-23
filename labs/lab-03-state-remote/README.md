# Lab 03: Remote State Management

## 🎯 Objective

Learn to configure and use remote state backends with state locking for team collaboration.

---

## 📋 Prerequisites

- AWS account with admin access
- Terraform installed
- Understanding of S3 and DynamoDB

---

## 📝 Steps

### Step 1: Create Backend Infrastructure

First, we need to create the S3 bucket and DynamoDB table for state management.

**backend-setup/main.tf**
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

# S3 bucket for Terraform state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "your-unique-terraform-state-bucket-12345"
}

# Enable versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name = "Terraform State Lock"
  }
}

output "s3_bucket_name" {
  value = aws_s3_bucket.terraform_state.bucket
}

output "dynamodb_table_name" {
  value = aws_dynamodb_table.terraform_locks.name
}
```

Apply this configuration:
```bash
cd backend-setup
terraform init
terraform apply
```

### Step 2: Create Main Project

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
  
  # Remote state configuration
  backend "s3" {
    bucket         = "your-unique-terraform-state-bucket-12345"
    key            = "lab-03/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = "us-west-2"
}

# Data source to demonstrate remote state querying
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "your-unique-terraform-state-bucket-12345"
    key    = "lab-03/network/terraform.tfstate"
    region = "us-west-2"
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "lab-03-vpc"
  }
}

# Subnet
resource "aws_subnet" "main" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-west-2a"
  
  tags = {
    Name = "lab-03-subnet"
  }
}

# Security group
resource "aws_security_group" "main" {
  name_prefix = "lab-03-sg"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "lab-03-sg"
  }
}
```

**outputs.tf**
```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "subnet_id" {
  description = "Subnet ID"
  value       = aws_subnet.main.id
}

output "security_group_id" {
  description = "Security Group ID"
  value       = aws_security_group.main.id
}
```

### Step 3: Initialize with Remote Backend

```bash
terraform init
```

**Observe:**
- Terraform asks if you want to migrate state
- State is now stored in S3
- Locking is enabled via DynamoDB

### Step 4: Apply Configuration

```bash
terraform apply
```

### Step 5: Verify Remote State

```bash
# Check S3 bucket
aws s3 ls s3://your-unique-terraform-state-bucket-12345/lab-03/

# View state content
aws s3 cp s3://your-unique-terraform-state-bucket-12345/lab-03/terraform.tfstate -

# Check DynamoDB for lock items
aws dynamodb scan --table-name terraform-state-lock
```

### Step 6: Test State Locking

Open two terminal windows:

**Terminal 1:**
```bash
terraform apply
# Keep this running...
```

**Terminal 2:**
```bash
terraform apply
# This should fail with lock error
```

### Step 7: State Commands

```bash
# List resources in state
terraform state list

# Show specific resource
terraform state show aws_vpc.main

# Pull state to local file
terraform state pull > my-state.tfstate

# View outputs
terraform output
```

### Step 8: State Import

Create a resource manually and import it:

```bash
# Create a security group manually in AWS Console
# Then import it:
terraform import aws_security_group.imported sg-xxxxxxxx
```

### Step 9: State Removal

```bash
# Remove resource from state (does NOT destroy!)
terraform state rm aws_security_group.main

# Now terraform plan will show it needs to be created
terraform plan
```

### Step 10: State Move

```bash
# Rename a resource in state
terraform state mv aws_security_group.main aws_security_group.web

# Update your .tf files to match the new name
```

---

## 🎓 What You Learned

1. **S3 Backend** - Store state remotely with versioning
2. **State Locking** - DynamoDB prevents concurrent modifications
3. **State Encryption** - SSE-S3 or KMS encryption
4. **State Commands** - list, show, pull, rm, mv, import
5. **Remote State Data** - Query state from other configurations
6. **State Security** - Never commit state, use encryption

---

## 📝 Key Concepts

### Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "my-state-bucket"
    key            = "path/to/state.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Backend Partial Configuration

```bash
# Provide backend config via CLI
terraform init \
  -backend-config="bucket=my-bucket" \
  -backend-config="key=prod.tfstate" \
  -backend-config="region=us-east-1"
```

### State Locking

```
User A: terraform apply
   └── Acquires lock in DynamoDB
   └── Makes changes
   └── Releases lock

User B: terraform apply (concurrent)
   └── Tries to acquire lock
   └── Fails: "Error acquiring the state lock"
```

### Remote State Data Source

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "state-bucket"
    key    = "network.tfstate"
    region = "us-west-2"
  }
}

# Use the data
resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.network.outputs.subnet_id
}
```

---

## 🚀 Challenge Exercises

1. **Set up state locking timeout** using `max_lock_timeout`
2. **Configure S3 bucket with KMS encryption**
3. **Create separate state files** for different components (network, compute)
4. **Use workspace-separated state**

---

## 🧹 Cleanup

```bash
# Destroy main resources
terraform destroy -auto-approve

# Destroy backend infrastructure
cd backend-setup
terraform destroy -auto-approve
```

---

[⬅️ Previous Lab](../lab-02-variables-locals/) | [Next Lab: Modules ➡️](../lab-04-modules-basics/)
