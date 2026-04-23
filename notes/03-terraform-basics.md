# 03 - Terraform Basics

> 📊 **Exam Weight:** 24% (Largest section!)

---

## HCL (HashiCorp Configuration Language)

HCL is the configuration language used by Terraform. It's designed to be human-readable and declarative.

### Basic Syntax

```hcl
# Block syntax
block_type "resource_type" "resource_name" {
  argument = "value"
  
  nested_block {
    argument = "value"
  }
}
```

### Configuration Building Blocks

| Element | Description | Example |
|---------|-------------|---------|
| **Blocks** | Containers for other content | `resource`, `provider`, `module` |
| **Arguments** | Assign values to names | `instance_type = "t2.micro"` |
| **Expressions** | Compute values | `var.region`, `local.name` |

---

## Core Configuration Components

### 1. Providers

Providers are plugins that Terraform uses to interact with cloud providers, SaaS providers, and other APIs.

```hcl
# Provider Configuration
provider "aws" {
  region     = "us-west-2"
  access_key = "my-access-key"
  secret_key = "my-secret-key"
}

# Required Providers Block (Terraform 0.13+)
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}
```

### 2. Resources

Resources represent infrastructure objects.

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name        = "ExampleInstance"
    Environment = "Dev"
  }
}

# Reference attributes
resource "aws_eip" "example" {
  instance = aws_instance.example.id
}
```

**Resource Naming Rules:**
- Must start with a letter
- Can contain letters, numbers, underscores, and hyphens
- Must be unique within the resource type

### 3. Data Sources

Query existing infrastructure or external data.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  owners = ["099720109477"]
}

# Use the data source
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}
```

### 4. Variables

Input variables parameterize configurations.

```hcl
# Variable Declaration
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Invalid instance type."
  }
}

variable "tags" {
  description = "Tags to apply"
  type        = map(string)
  default     = {}
}

variable "availability_zones" {
  description = "List of AZs"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b"]
}
```

**Variable Types:**
- `string`
- `number`
- `bool`
- `list(<TYPE>)`
- `set(<TYPE>)`
- `map(<TYPE>)`
- `object({<ATTR_NAME> = <TYPE>, ...})`
- `tuple([<TYPE>, ...])`
- `any` (default)

### 5. Locals

Local values assign a name to an expression.

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
  }
  
  instance_name = "${var.project}-${var.environment}-instance"
}

# Usage
resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = merge(local.common_tags, {
    Name = local.instance_name
  })
}
```

**Variables vs Locals:**
| Feature | Variables | Locals |
|---------|-----------|--------|
| Set at runtime | Yes | No |
| Can have defaults | Yes | No |
| Computed values | No | Yes |
| Purpose | Input parameters | Intermediate calculations |

### 6. Outputs

Output values expose information about infrastructure.

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.example.id
  sensitive   = false
}

output "private_ip" {
  description = "Private IP of the instance"
  value       = aws_instance.example.private_ip
  sensitive   = true  # Masked in logs
}
```

---

## Expressions and Functions

### String Interpolation

```hcl
locals {
  full_name = "${var.first_name} ${var.last_name}"
  # Or with newer syntax
  full_name_new = "${var.first_name} ${var.last_name}"
}
```

### Conditionals

```hcl
resource "aws_instance" "example" {
  count = var.create_instance ? 1 : 0
  
  ami           = var.ami_id
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
}
```

### Built-in Functions

```hcl
# String functions
upper("hello")           # "HELLO"
lower("WORLD")           # "world"
substr("hello", 0, 4)    # "hell"
format("Hello %s", "World")  # "Hello World"

# Collection functions
length(["a", "b", "c"])  # 3
merge({a = 1}, {b = 2})  # {a = 1, b = 2}
keys({a = 1, b = 2})     # ["a", "b"]
values({a = 1, b = 2})   # [1, 2]
lookup({a = 1}, "b", 0)  # 0 (default)

# Numeric functions
max(1, 2, 3)             # 3
min(1, 2, 3)             # 1
ceil(1.2)                # 2
floor(1.9)               # 1

# File functions
file("script.sh")                 # Read file contents
filebase64("binary.dat")          # Read and base64 encode
templatefile("template.tpl", {name = "test"})  # Process template
```

---

## Meta-Arguments

Meta-arguments change resource behavior.

### `depends_on`

Explicitly specify dependencies.

```hcl
resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = "t2.micro"
  
  depends_on = [aws_iam_role_policy.example]
}
```

### `count`

Create multiple instances of a resource.

```hcl
resource "aws_instance" "web" {
  count = 3
  
  ami           = var.ami_id
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}

# Access individual instances
aws_instance.web[0]
aws_instance.web[1]
aws_instance.web[2]
```

### `for_each`

Iterate over a map or set of strings.

```hcl
locals {
  instances = {
    web-1 = "t2.micro"
    web-2 = "t2.small"
    web-3 = "t2.medium"
  }
}

resource "aws_instance" "web" {
  for_each = local.instances
  
  ami           = var.ami_id
  instance_type = each.value
  
  tags = {
    Name = each.key
  }
}

# Access individual instances
aws_instance.web["web-1"]
aws_instance.web["web-2"]
```

**count vs for_each:**
| Use Case | Use |
|----------|-----|
| Number of identical resources | `count` |
| Different configurations per resource | `for_each` |
| Resources based on a collection | `for_each` |

### `provider`

Specify a non-default provider.

```hcl
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "east"
  region = "us-east-1"
}

resource "aws_instance" "west" {
  provider = aws.west
  
  ami           = "ami-west"
  instance_type = "t2.micro"
}

resource "aws_instance" "east" {
  provider = aws.east
  
  ami           = "ami-east"
  instance_type = "t2.micro"
}
```

### `lifecycle`

Customize resource lifecycle.

```hcl
resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags]
    replace_triggered_by  = [aws_kms_key.example]
  }
}
```

**Lifecycle Arguments:**
- `create_before_destroy`: Create new resource before destroying old
- `prevent_destroy`: Prevent accidental destruction
- `ignore_changes`: Ignore specific attribute changes
- `replace_triggered_by`: Replace when other resources change

---

## Dynamic Blocks

Create nested blocks dynamically.

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
  ]
}

resource "aws_security_group" "example" {
  name = "example"
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

---

## Exam Tips

1. **Variable types matter** - Know the difference between `list`, `set`, `map`, `object`, `tuple`
2. **Locals vs Variables** - Locals are for computed values, variables are for inputs
3. **count vs for_each** - Use `for_each` when resources have different configs
4. **Meta-arguments** - Know all five: `depends_on`, `count`, `for_each`, `provider`, `lifecycle`
5. **Sensitive outputs** - Mark sensitive data to hide in logs
6. **State file** - Contains sensitive data; secure it properly

---

[⬅️ Previous: Terraform's Purpose](02-terraform-purpose.md) | [Next: CLI Commands ➡️](04-cli-commands.md)
