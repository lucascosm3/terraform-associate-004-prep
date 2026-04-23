# 04 - Domain 4: Terraform Configuration

> Exam Weight: ~22% (Largest domain!) | Tested on Terraform 1.12

---

## Objectives

- [x] **4a:** Use and differentiate resource and data blocks
- [x] **4b:** Refer to resource attributes and create cross-resource references
- [x] **4c:** Use variables and outputs
- [x] **4d:** Understand and use complex types
- [x] **4e:** Write dynamic configuration using expressions and functions
- [x] **4f:** Define resource dependencies in configuration
- [x] **4g:** Validate configuration using custom conditions (preconditions, postconditions, checks)
- [x] **4h:** Understand best practices for managing sensitive data, including secrets management with Vault

---

## 4a. Resource and Data Blocks

### Resource Blocks

Resources are the primary building block — they define infrastructure objects that Terraform will manage.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "WebServer"
  }
}
```

**Resource address format:** `resource_type.resource_name`
- Example: `aws_instance.web`

**Resource naming rules:**
- Must start with a letter or underscore
- Can contain letters, digits, underscores, and hyphens
- Must be unique within the resource type in a module

### Data Blocks

Data sources fetch information about existing infrastructure or compute values. They are **read-only** — they do not create or modify resources.

```hcl
# Read an existing AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use the data source in a resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}
```

### Resource vs Data Source Comparison

| Aspect | Resource | Data Source |
|--------|----------|-------------|
| **Action** | Creates, updates, or deletes | Read-only: queries existing |
| **Type** | `resource "TYPE" "NAME" {}` | `data "TYPE" "NAME" {}` |
| **Reference** | `TYPE.NAME.ATTRIBUTE` | `data.TYPE.NAME.ATTRIBUTE` |
| **State** | Managed in state | Read during plan/apply |
| **Purpose** | Define desired infrastructure | Query existing infrastructure |

### Common Data Sources

```hcl
# Get current AWS account info
data "aws_caller_identity" "current" {}

# Get current region
data "aws_region" "current" {}

# Get availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Read an existing VPC by ID
data "aws_vpc" "existing" {
  id = "vpc-12345678"
}

# Read an existing VPC by filter
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
}

# Read a Terraform remote state
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### Data Source Filtering

```hcl
data "aws_subnet" "selected" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }

  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
}
```

---

## 4b. Resource Attributes and Cross-Resource References

### Accessing Resource Attributes

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Reference attributes of the resource
output "instance_id" {
  value = aws_instance.web.id
}

output "instance_arn" {
  value = aws_instance.web.arn
}

output "instance_private_ip" {
  value = aws_instance.web.private_ip
}

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}
```

### Cross-Resource References (Implicit Dependencies)

When one resource references another, Terraform automatically creates a dependency:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id       # Implicit dependency
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id  # Implicit dependency

  tags = {
    VpcId = aws_vpc.main.id             # Also creates dependency
  }
}
```

**Terraform creates resources in dependency order:**
1. `aws_vpc.main` (no dependencies)
2. `aws_subnet.public` (depends on `aws_vpc.main`)
3. `aws_instance.web` (depends on `aws_subnet.public` and `aws_vpc.main`)

### Splat Expressions

```hcl
# Get all private IPs from a list of instances
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Splat expression — get all IDs
output "instance_ids" {
  value = aws_instance.web[*].id
}

# Full splat expression (legacy, same result)
output "instance_ids_legacy" {
  value = aws_instance.web.*.id
}
```

---

## 4c. Variables and Outputs

### Input Variables

Input variables parameterize configurations, making them reusable and flexible.

```hcl
# Basic variable
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

# Variable with validation
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Variable with sensitive data
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

# Variable with nullable and custom default
variable "enable_monitoring" {
  description = "Enable monitoring"
  type        = bool
  default     = true
  nullable    = false
}
```

### Variable Types

```hcl
variable "string_var" {
  type    = string
  default = "hello"
}

variable "number_var" {
  type    = number
  default = 42
}

variable "bool_var" {
  type    = bool
  default = true
}

variable "list_var" {
  type    = list(string)
  default = ["a", "b", "c"]
}

variable "set_var" {
  type    = set(string)
  default = ["a", "b", "c"]  # Order not guaranteed
}

variable "map_var" {
  type    = map(string)
  default = {
    key1 = "value1"
    key2 = "value2"
  }
}

variable "object_var" {
  type = object({
    name = string
    age  = number
  })
  default = {
    name = "Alice"
    age  = 30
  }
}

variable "tuple_var" {
  type    = tuple([string, number, bool])
  default = ["hello", 42, true]
}
```

### Variable Precedence (highest to lowest)

1. `-var` and `-var-file` command-line arguments
2. `auto.tfvars` files (alphabetical order)
3. `terraform.tfvars` file
4. Environment variables (`TF_VAR_<name>`)
5. Default values in variable declarations

```bash
# Override via command line (highest precedence)
terraform apply -var="instance_type=t2.large"

# Override via .tfvars file
terraform apply -var-file="production.tfvars"

# Environment variable
export TF_VAR_instance_type="t2.large"
terraform apply
```

### Locals

Local values assign names to expressions for reuse within a module.

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
  }

  instance_name = "${var.project}-${var.environment}-instance"

  # Computed values
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

resource "aws_instance" "web" {
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
| Set at runtime | Yes (CLI, .tfvars, env) | No |
| Can have defaults | Yes | No |
| Computed from other values | No | Yes |
| Purpose | Input parameters | Intermediate calculations |
| Scope | Module (input from outside) | Module (internal reuse) |

### Output Values

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.web.public_ip
}

output "db_connection_string" {
  description = "Database connection string"
  value       = "postgres://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}"
  sensitive   = true
}
```

**Key points about outputs:**
- `sensitive = true` — masks the value in CLI output (but it's still in state!)
- `description` — documents what the output provides
- Outputs from modules are accessed as `module.<name>.<output>`

---

## 4d. Complex Types

### Type Summary

| Type | Description | Example |
|------|-------------|---------|
| `string` | Text value | `"hello"` |
| `number` | Numeric value | `42`, `3.14` |
| `bool` | Boolean value | `true`, `false` |
| `list(<TYPE>)` | Ordered sequence | `["a", "b", "c"]` |
| `set(<TYPE>)` | Unique unordered collection | `["a", "b", "c"]` |
| `map(<TYPE>)` | Key-value pairs (same type) | `{a = "1", b = "2"}` |
| `object({...})` | Structured data with typed attributes | `{name = "a", port = 80}` |
| `tuple([...])` | Fixed-length sequence of typed elements | `["a", 80, true]` |
| `any` | Accepts any type | (convenience) |

### Object Type

```hcl
variable "instance_config" {
  type = object({
    name          = string
    instance_type = string
    ami           = string
    tags          = map(string)
    enable_monitoring = optional(bool, false)
  })

  default = {
    name              = "web-server"
    instance_type     = "t2.micro"
    ami               = "ami-0c55b159cbfafe1f0"
    tags              = { Environment = "dev" }
    enable_monitoring = false
  }
}

resource "aws_instance" "web" {
  ami           = var.instance_config.ami
  instance_type = var.instance_config.instance_type

  tags = merge(var.instance_config.tags, {
    Name = var.instance_config.name
  })
}
```

### Optional Attributes in Objects (Terraform 1.3+)

```hcl
variable "server_config" {
  type = object({
    name        = string
    port        = optional(number, 8080)
    description = optional(string, "Default description")
  })

  default = {
    name = "my-server"
    # port and description use their default values
  }
}
```

### Tuple Type

```hcl
variable "mixed_values" {
  type    = tuple([string, number, bool])
  default = ["hello", 42, true]
}
```

### List vs Set

```hcl
# List — ordered, allows duplicates
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Set — unordered, no duplicates
variable "unique_cidrs" {
  type    = set(string)
  default = ["10.0.0.0/24", "10.0.1.0/24"]
}
```

### Map vs Object

```hcl
# Map — all values must be the same type
variable "instance_types" {
  type    = map(string)
  default = {
    web  = "t2.micro"
    db   = "t3.medium"
    app  = "t2.small"
  }
}

# Object — values can be different types
variable "config" {
  type = object({
    instance_type = string
    count         = number
    enable        = bool
  })
  default = {
    instance_type = "t2.micro"
    count         = 3
    enable        = true
  }
}
```

---

## 4e. Expressions and Functions

### String Interpolation

```hcl
locals {
  resource_name = "${var.project}-${var.environment}-instance"
  
  # Newer syntax (same result)
  resource_name_v2 = "${var.project}-${var.environment}-instance"
}
```

### Conditional Expressions

```hcl
# condition ? true_val : false_val
resource "aws_instance" "web" {
  count         = var.create_instance ? 1 : 0
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
}
```

### For Expressions

```hcl
# Transform a list
locals {
  instance_names = [for s in var.instances : "${s}-instance"]
  # ["web-instance", "db-instance", "app-instance"]
}

# Transform a map
locals {
  upper_tags = { for k, v in var.tags : upper(k) => upper(v) }
}

# Filter with for
locals {
  private_subnets = [for s in var.subnets : s if s.type == "private"]
}

# Group by
locals {
  instances_by_type = {
    for inst in var.instances : inst.type => inst.name...
  }
}
```

### Splat Expressions

```hcl
# Get all IDs from a list of resources
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

output "all_ids" {
  value = aws_instance.web[*].id
}

output "all_ips" {
  value = aws_instance.web[*].private_ip
}
```

### Built-in Functions

#### String Functions

```hcl
upper("hello")                                    # "HELLO"
lower("WORLD")                                     # "world"
title("hello world")                                # "Hello World"
substr("hello", 0, 4)                              # "hell"
replace("hello world", "world", "terraform")       # "hello terraform"
strrev("hello")                                     # "olleh"
join(",", ["a", "b", "c"])                          # "a,b,c"
split(",", "a,b,c")                                 # ["a", "b", "c"]
format("Hello %s, you have %d messages", "Alice", 5)
formatlist("instance-%s", ["a", "b", "c"])         # ["instance-a", "instance-b", "instance-c"]
trimsuffix("hello.txt", ".txt")                    # "hello"
trimprefix("hello.txt", "hello.")                  # "txt"
trimspace("  hello  ")                              # "hello"
chomp("hello\n")                                    # "hello"
indent(4, "line1\nline2")                           # "    line1\n    line2"
coalesce("", "", "default")                         # "default"
```

#### Numeric Functions

```hcl
max(1, 2, 3)       # 3
min(1, 2, 3)       # 1
ceil(1.2)           # 2
floor(1.9)          # 1
abs(-5)             # 5
parseint("100", 10) # 100
```

#### Collection Functions

```hcl
length(["a", "b", "c"])                            # 3
element(["a", "b", "c"], 1)                        # "b" (0-indexed)
index(["a", "b", "c"], "b")                         # 1
contains(["a", "b", "c"], "b")                      # true
distinct(["a", "a", "b"])                           # ["a", "b"]
flatten([["a"], ["b", "c"]])                         # ["a", "b", "c"]
reverse(["a", "b", "c"])                            # ["c", "b", "a"]
slice(["a", "b", "c", "d"], 1, 3)                   # ["b", "c"]
sort(["c", "a", "b"])                               # ["a", "b", "c"]
keys({a = 1, b = 2})                                # ["a", "b"]
values({a = 1, b = 2})                              # [1, 2]
lookup({a = 1}, "b", 0)                             # 0 (default value)
lookup({a = 1}, "a", 0)                             # 1
merge({a = 1}, {b = 2})                             # {a = 1, b = 2}
merge({a = 1}, {a = 2})                             # {a = 2} (overwrites)
zipmap(["a", "b"], [1, 2])                           # {a = 1, b = 2}
setsubtract(["a", "b", "c"], ["b"])                  # ["a", "c"]
setintersection(["a", "b"], ["b", "c"])              # ["b"]
setunion(["a", "b"], ["b", "c"])                     # ["a", "b", "c"]
toset(["a", "b", "a"])                               # ["a", "b"] (unique)
```

#### Type Conversion Functions

```hcl
tostring(1)           # "1"
tonumber("42")        # 42
tobool("true")        # true
can(tonumber("abc"))  # false
try(tonumber("abc"), 0) # 0 (fallback on error)
```

#### File Functions

```hcl
file("script.sh")                       # Read file as string
filebase64("image.png")                 # Base64 encode file
templatefile("user_data.sh", {          # Process template
  name = "web-server",
  port = 8080
})
fileset("configs/", "*.conf")           # List matching files
path.module                            # Current module directory
path.root                              # Root module directory
path.cwd                               # Current working directory
```

#### Encoding Functions

```hcl
base64encode("hello")          # "aGVsbG8="
base64decode("aGVsbG8=")       # "hello"
jsonencode({a = 1, b = 2})    # '{"a":1,"b":2}'
jsondecode('{"a":1}')           # {a = 1}
urlencode("hello world")        # "hello+world"
yamlencode({key = "value"})    # "key: value\n"
yamldecode("key: value")       # {key = "value"}
```

#### Date and Time Functions

```hcl
timestamp()                              # "2024-01-15T10:30:00Z"
formatdate("YYYY-MM-DD", timestamp())   # "2024-01-15"
timeadd(timestamp(), "24h")              # Add 24 hours
```

#### Network Functions

```hcl
cidrhost("10.0.0.0/16", 5)              # "10.0.0.5"
cidrnetmask("10.0.0.0/16")              # "255.255.0.0"
cidrsubnet("10.0.0.0/16", 8, 1)         # "10.0.1.0/24"
cidrsubnets("10.0.0.0/16", 8, 8, 8)    # ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
```

### Dynamic Blocks

Dynamic blocks generate nested blocks from collections:

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

resource "aws_security_group" "web" {
  name = "web-sg"

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

**Custom iterator name:**

```hcl
dynamic "tag" {
  for_each = var.tags
  iterator = item
  content {
    key   = item.key
    value = item.value
  }
}
```

**Conditional dynamic block** (include only when variable is true):

```hcl
variable "enable_logging" {
  type    = bool
  default = true
}

resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"

  dynamic "logging" {
    for_each = var.enable_logging ? [1] : []
    content {
      target_bucket = "logs-bucket"
      target_prefix = "logs/"
    }
  }
}
```

---

## 4f. Resource Dependencies

### Implicit Dependencies

Terraform automatically detects dependencies when one resource references another:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id    # Implicit dependency
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id  # Implicit dependency
  ami       = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

### Explicit Dependencies (`depends_on`)

Use `depends_on` when Terraform cannot detect a dependency automatically:

```hcl
resource "aws_iam_role_policy" "example" {
  name = "example"
  role = aws_iam_role.example.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "s3:GetObject"
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  depends_on = [
    aws_iam_role_policy.example
  ]
}
```

**When to use `depends_on`:**
- When a resource needs another resource to be fully created before it works
- When there's no attribute reference to create an implicit dependency
- Example: EC2 instance needs IAM role policy to be active

### Lifecycle Meta-Arguments

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags, ami]
    replace_triggered_by  = [aws_kms_key.example]
  }
}
```

| Argument | Description |
|----------|-------------|
| `create_before_destroy` | Create new resource before destroying old (default is destroy-then-create) |
| `prevent_destroy` | Prevent accidental destruction; `terraform destroy` will fail |
| `ignore_changes` | Ignore changes to specified attributes |
| `replace_triggered_by` | Replaces resource when specified items change |

```hcl
# Ignore specific attributes
lifecycle {
  ignore_changes = [
    tags,
    ami,
  ]
}

# Ignore all attributes
lifecycle {
  ignore_changes = all
}
```

### `count` and `for_each` Meta-Arguments

#### `count`

```hcl
variable "instance_count" {
  default = 3
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-${count.index + 1}"
  }
}

# Access individual instances
output "first_instance_id" {
  value = aws_instance.web[0].id
}

# Access all instances
output "all_instance_ids" {
  value = aws_instance.web[*].id
}
```

#### `for_each`

```hcl
locals {
  instances = {
    web-1 = { type = "t2.micro", az = "us-east-1a" }
    web-2 = { type = "t2.small", az = "us-east-1b" }
    web-3 = { type = "t2.medium", az = "us-east-1c" }
  }
}

resource "aws_instance" "web" {
  for_each = local.instances

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value.type
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}

# Access individual instances
output "web1_id" {
  value = aws_instance.web["web-1"].id
}
```

**count vs for_each:**

| Feature | `count` | `for_each` |
|---------|---------|-------------|
| Input type | Number | Map or set of strings |
| Resource address | `resource.type[index]` | `resource.type["key"]` |
| Best for | Identical resources | Different configurations per resource |
| Removing items | Shifts indexes (dangerous) | Only removes the removed item |

**Best practice:** Prefer `for_each` over `count` when resources might be added or removed, because removing an item from the middle of a `count` list shifts all subsequent indexes.

### `moved` Block (Terraform 1.1+)

The `moved` block tells Terraform that a resource has been renamed or moved, ensuring state is updated without destroying and recreating:

```hcl
# Old resource (removed from config)
# resource "aws_instance" "old_web" { ... }

# New resource (in config)
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Moved block — tells Terraform the resource was renamed
moved {
  from = aws_instance.old_web
  to   = aws_instance.web
}
```

**Key points:**
- No resources are destroyed or recreated
- The state entry is simply moved to the new address
- After applying, the `moved` block can remain in config or be removed

### `removed` Block (Terraform 1.7+)

The `removed` block removes a resource from state without destroying the real infrastructure:

```hcl
# This resource will be removed from Terraform management
# but NOT destroyed in the cloud
removed {
  from = aws_instance.deprecated

  lifecycle {
    destroy = false  # Set to true to destroy the resource
  }
}
```

---

## 4g. Custom Conditions (Preconditions, Postconditions, Checks)

### Preconditions

Preconditions check that a condition is true BEFORE a resource is created or updated. If the condition fails, the operation is blocked.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = var.instance_type != "t2.nano"
      error_message = "t2.nano instances are not supported for production workloads."
    }

    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "Selected AMI must be x86_64 architecture."
    }
  }
}
```

### Postconditions

Postconditions check that a condition is true AFTER a resource is created or updated. They are useful for validating output.

```hcl
data "aws_ebs_volume" "example" {
  filter {
    name   = "volume-id"
    values = [aws_instance.web.root_block_device[0].volume_id]
  }
}

resource "aws_ebs_snapshot" "example" {
  volume_id = data.aws_ebs_volume.example.id

  lifecycle {
    postcondition {
      condition     = self.volume_size >= 8
      error_message = "Snapshot volume size must be at least 8 GB."
    }
  }
}
```

**Note:** Postconditions use `self` to reference the resource they are defined in.

### `check` Blocks (Terraform 1.5+)

Check blocks evaluate conditions independently of resources. They are useful for validating data sources or asserting assumptions.

```hcl
check "ami_exists" {
  assert {
    condition     = data.aws_ami.ubuntu.id != ""
    error_message = "Could not find a valid Ubuntu AMI."
  }

  assert {
    condition     = data.aws_ami.ubuntu.architecture == "x86_64"
    error_message = "AMI architecture must be x86_64."
  }
}
```

```hcl
# Check that VPC CIDR is in the allowed range
check "vpc_cidr_valid" {
  assert {
    condition     = tonumber(regex("[0-9]+", aws_vpc.main.cidr_block)) < 172
    error_message = "VPC CIDR must be in the private range."
  }
}
```

**Check blocks:**
- Run during `terraform plan`
- Can reference data sources and resources
- Do not create any infrastructure
- Multiple `assert` blocks can be in a single `check`
- Named check blocks appear in plan/apply output

### Preconditions vs Postconditions vs Checks

| Feature | `precondition` | `postcondition` | `check` |
|---------|---------------|-----------------|---------|
| **When** | Before resource action | After resource action | During plan |
| **Where** | Inside `lifecycle {}` | Inside `lifecycle {}` | Top-level block |
| **References** | `var`, `data`, `local` | `self`, other resources | `data`, `var`, `local` |
| **Purpose** | Validate inputs before changes | Validate outputs after changes | General assertions |
| **Failure** | Blocks the operation | Blocks the operation | Shows warning/error |

---

## 4h. Sensitive Data and Secrets Management with Vault

### Sensitive Variables and Outputs

```hcl
# Sensitive input variable
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

# Sensitive output
output "connection_string" {
  description = "Database connection string"
  value       = "postgres://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}"
  sensitive   = true
}
```

**Key behavior:**
- Sensitive values are **redacted in CLI output**
- Sensitive values are **still stored in state files** (this is important!)
- Marking an output as `sensitive = true` hides it in the CLI but it can still be accessed via `terraform output`
- If a variable is used in a non-sensitive context, Terraform will warn

### Sensitive in State

**WARNING:** State files contain ALL attribute values, including those marked as sensitive. This is why state files must be secured:

1. Enable encryption on remote state backends
2. Restrict access to state files
3. Use a secrets manager for truly sensitive values

```hcl
# S3 backend with encryption
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true  # Encrypt state at rest
    dynamodb_table = "terraform-locks"
  }
}
```

### The `nonsensitive()` Function

```hcl
# Temporarily remove sensitive marking (use with caution!)
output "debug_info" {
  value = nonsensitive(var.db_password)
}
```

**Warning:** Only use `nonsensitive()` in development/debugging. Never in production outputs.

### Vault Integration for Secrets

HashiCorp Vault is the recommended way to manage secrets in Terraform workflows.

#### Vault Provider

```hcl
terraform {
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "~> 3.0"
    }
  }
}

provider "vault" {
  address = "https://vault.example.com:8200"
  token   = var.vault_token
}
```

#### Reading Secrets from Vault

```hcl
# Read a secret from Vault KV v2
data "vault_kv_secret_v2" "db_credentials" {
  mount = "secret"
  name  = "database/production"
}

resource "aws_db_instance" "main" {
  engine               = "postgres"
  instance_class       = "db.t3.micro"
  username             = data.vault_kv_secret_v2.db_credentials.data["username"]
  password             = data.vault_kv_secret_v2.db_credentials.data["password"]
  skip_final_snapshot  = true
}
```

#### Reading Dynamic Secrets from Vault

```hcl
# Generate dynamic AWS credentials from Vault
data "vault_aws_access_credentials" "creds" {
  backend = "aws"
  role    = "terraform-role"
  type    = "sts"
}

provider "aws" {
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
  region     = "us-east-1"
}
```

#### Vault Generic Secret

```hcl
# Read from Vault KV v1
data "vault_generic_secret" "api_key" {
  path = "secret/data/api-keys/external-service"
}

resource "aws_lambda_function" "api" {
  # ...
  environment {
    variables = {
      API_KEY = data.vault_generic_secret.api_key.data["api_key"]
    }
  }
}
```

### Other Secrets Management Approaches

| Approach | Description | Best For |
|----------|-------------|----------|
| **Vault** | Dynamic and static secrets, rotation, audit | Production, enterprise |
| **AWS Secrets Manager** | Secrets with automatic rotation | AWS-centric environments |
| **AWS SSM Parameter Store** | Simple parameter storage | AWS, simpler use cases |
| **Environment variables** | `TF_VAR_<name>` | Development, CI/CD |
| **`.tfvars` files (gitignored)** | Separate variable files | Small teams |
| **SOPS** | Encrypted secrets files | GitOps workflows |

```hcl
# AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "prod/database"
}

resource "aws_db_instance" "main" {
  username = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)["username"]
  password = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)["password"]
}

# AWS SSM Parameter Store
data "aws_ssm_parameter" "db_password" {
  name            = "/prod/database/password"
  with_decryption = true
}
```

### Best Practices for Sensitive Data

1. **Never hardcode secrets** in `.tf` files
2. **Use Vault or cloud-native secrets managers** for production
3. **Mark variables and outputs as `sensitive`** to prevent accidental exposure
4. **Encrypt state files at rest** — use `encrypt = true` for S3 backends
5. **Restrict state file access** — use IAM policies and backend access controls
6. **Add `.tfvars` to `.gitignore`** if using variable files for secrets
7. **Use `TF_VAR_<name>` environment variables** in CI/CD
8. **Rotate secrets regularly** — Vault and AWS Secrets Manager support automatic rotation

---

## Exam Tips

1. **Resources create, data sources read** — know the syntax difference (`resource` vs `data`)
2. **Implicit dependencies** — referencing `aws_vpc.main.id` creates a dependency automatically
3. **Variable precedence** — CLI flags > auto.tfvars > terraform.tfvars > env vars > defaults
4. **`for_each` over `count`** — prefer `for_each` for resources that might be removed
5. **`sensitive = true`** — only hides in output; state still contains the value
6. **`moved` block** — rename resources without destroying them
7. **`removed` block** — remove from state without destroying real infrastructure
8. **Preconditions** — validate BEFORE changes; postconditions validate AFTER
9. **`check` blocks** — independent assertions during plan
10. **State files contain secrets** — always use encrypted remote backends
11. **Vault integration** — know how to read secrets with the Vault provider
12. **`nonsensitive()`** — use only for debugging; never in production

---

## Quick Review Questions

1. **Q:** What is the difference between a resource and a data source?
   **A:** Resources create and manage infrastructure; data sources read existing infrastructure (read-only).

2. **Q:** How does Terraform determine resource creation order?
   **A:** Through implicit dependencies (attribute references) and explicit `depends_on` declarations.

3. **Q:** What is the variable precedence order (highest to lowest)?
   **A:** CLI `-var`/`-var-file` > `auto.tfvars` files > `terraform.tfvars` > `TF_VAR_` env vars > defaults.

4. **Q:** When should you use `for_each` instead of `count`?
   **A:** When resources have different configurations or when items might be removed from the middle of the list.

5. **Q:** What does the `moved` block do?
   **A:** It tells Terraform that a resource has been renamed or moved, ensuring the state is updated without destroying and recreating the resource.

6. **Q:** What does the `removed` block do?
   **A:** It removes a resource from Terraform management without destroying the actual infrastructure.

7. **Q:** What is the difference between a precondition and a postcondition?
   **A:** Preconditions are checked before a resource action; postconditions are checked after.

8. **Q:** Does marking a variable as `sensitive = true` prevent it from appearing in state?
   **A:** No! It only redacts the value in CLI output. The value is still stored in the state file.

9. **Q:** How can you integrate Vault with Terraform for secrets?
   **A:** Use the `hashicorp/vault` provider to read secrets via `vault_kv_secret_v2` or `vault_generic_secret` data sources.

10. **Q:** What is a `check` block?
    **A:** A top-level block that evaluates conditions during `terraform plan`, independent of any specific resource.

---

[⬅️ Previous: Core Workflow](03-core-workflow.md) | [Next: Modules ➡️](05-modules.md)