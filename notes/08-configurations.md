# 08 - Read, Generate, Modify Configurations

> 📊 **Exam Weight:** 6%

---

## Terraform Built-in Functions

Terraform includes many built-in functions for manipulating data.

### String Functions

```hcl
# Basic string manipulation
upper("hello")                    # "HELLO"
lower("WORLD")                    # "world"
title("hello world")              # "Hello World"

# Substring and replacement
substr("hello", 0, 4)             # "hell"
substr("hello", -2, 2)            # "lo"
replace("hello world", "world", "terraform")  # "hello terraform"
strrev("hello")                   # "olleh"

# Formatting
format("Hello %s, you have %d messages", "Alice", 5)
# "Hello Alice, you have 5 messages"

formatlist("instance-%s", ["a", "b", "c"])
# ["instance-a", "instance-b", "instance-c"]

# Joining and splitting
join(",", ["a", "b", "c"])         # "a,b,c"
split(",", "a,b,c")               # ["a", "b", "c"]

# Trimming
trimsuffix("hello.txt", ".txt")   # "hello"
trimprefix("hello.txt", "hello.") # "txt"
trimspace("  hello  ")             # "hello"

# Other useful functions
chomp("hello\n")                   # "hello"
indent(2, "line1\nline2")          # "  line1\n  line2"
```

### Numeric Functions

```hcl
# Basic math
max(1, 2, 3)                      # 3
min(1, 2, 3)                      # 1
ceil(1.2)                         # 2
floor(1.9)                        # 1
abs(-5)                           # 5
signum(-5)                        # -1
signum(5)                         # 1
signum(0)                         # 0

# Parsing
parseint("100", 10)               # 100
```

### Collection Functions

```hcl
# List functions
length(["a", "b", "c"])           # 3
element(["a", "b", "c"], 1)       # "b" (0-indexed)
index(["a", "b", "c"], "b")       # 1
contains(["a", "b", "c"], "b")    # true
distinct(["a", "a", "b"])         # ["a", "b"]
flatten([["a"], ["b", "c"]])      # ["a", "b", "c"]
reverse(["a", "b", "c"])          # ["c", "b", "a"]
slice(["a", "b", "c", "d"], 1, 3) # ["b", "c"]
sort(["c", "a", "b"])             # ["a", "b", "c"]

# Map functions
keys({a = 1, b = 2})              # ["a", "b"]
values({a = 1, b = 2})            # [1, 2]
lookup({a = 1}, "b", 0)           # 0 (default)
lookup({a = 1}, "a", 0)           # 1
merge({a = 1}, {b = 2})           # {a = 1, b = 2}
merge({a = 1}, {a = 2})           # {a = 2} (overwrites)

# Zipmap - create map from keys and values
zipmap(["a", "b"], [1, 2])        # {a = 1, b = 2}

# Set functions
setsubtract(["a", "b", "c"], ["b"])     # ["a", "c"]
setintersection(["a", "b"], ["b", "c"]) # ["b"]
setunion(["a", "b"], ["b", "c"])        # ["a", "b", "c"]
```

### Type Conversion Functions

```hcl
# Type checking
tostring(1)                       # "1"
tonumber("1")                     # 1
tobool("true")                    # true

# Type checking functions
can(tostring(1))                  # true
can(tostring([]))                 # false

try(
  tonumber("not-a-number"),
  0  # fallback
)                                 # 0
```

### File and Filesystem Functions

```hcl
# Read files
file("script.sh")                 # Read file as string
filemd5("file.txt")               # MD5 hash of file
filesha256("file.txt")            # SHA256 hash of file
filebase64("image.png")           # Base64 encode file
filebase64sha256("file.txt")      # SHA256 of base64 content

# Template files
templatefile("user_data.sh", {
  name = "web-server"
  port = 8080
})

# Path references
path.module                       # Path to current module
path.root                         # Path to root module
path.cwd                          # Current working directory

# Directory operations
fileset("configs/", "*.conf")     # List files matching pattern
```

### Date and Time Functions

```hcl
timestamp()                       # Current UTC time: "2023-01-01T00:00:00Z"
formatdate("YYYY-MM-DD", timestamp())  # "2023-01-01"
timeadd(timestamp(), "24h")       # Add 24 hours
```

### IP Network Functions

```hcl
cidrhost("10.0.0.0/16", 5)        # "10.0.0.5"
cidrnetmask("10.0.0.0/16")        # "255.255.0.0"
cidrsubnet("10.0.0.0/16", 8, 1)   # "10.0.1.0/24"
cidrsubnets("10.0.0.0/16", 8, 8, 8)  # ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
```

### Encoding Functions

```hcl
base64encode("hello")             # "aGVsbG8="
base64decode("aGVsbG8=")          # "hello"
jsonencode({a = 1, b = 2})        # '{"a":1,"b":2}'
jsondecode('{"a":1}')             # {a = 1}
urlencode("hello world")          # "hello+world"
```

---

## Dynamic Blocks

Dynamic blocks create nested blocks based on collections.

### Basic Dynamic Block

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP"
    },
    {
      port        = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS"
    }
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
      description = ingress.value.description
    }
  }
}
```

### Conditional Dynamic Blocks

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

### Iterator Alternative

```hcl
dynamic "tag" {
  for_each = var.tags
  iterator = item  # Use custom name instead of "tag"
  content {
    key                 = item.key
    value               = item.value
    propagate_at_launch = true
  }
}
```

---

## Data Sources

Data sources fetch information about existing infrastructure.

### Built-in Data Sources

```hcl
# Current AWS caller identity
data "aws_caller_identity" "current" {}
output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

# AWS region
data "aws_region" "current" {}
output "region" {
  value = data.aws_region.current.name
}

# Availability Zones
data "aws_availability_zones" "available" {
  state = "available"
}
output "azs" {
  value = data.aws_availability_zones.available.names
}

# Get AMI
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
```

### Data Source Usage

```hcl
# Reference existing VPC
data "aws_vpc" "existing" {
  id = "vpc-12345678"
}

# Reference by filter
data "aws_subnet" "selected" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
  
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
}

resource "aws_instance" "example" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  subnet_id     = data.aws_subnet.selected.id
}
```

---

## Resource Attributes and References

### Resource Attributes

Every resource exports attributes that can be referenced:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Reference attributes
aws_instance.web.id
aws_instance.web.arn
aws_instance.web.private_ip
aws_instance.web.public_ip
aws_instance.web.availability_zone
```

### Self References

```hcl
resource "aws_security_group" "example" {
  name = "example"
  
  # Self-reference during creation
  ingress {
    from_port = 0
    to_port   = 0
    protocol  = -1
    self      = true  # Allow traffic from resources with this security group
  }
}
```

### Implicit Dependencies

```hcl
# Terraform automatically detects dependencies
resource "aws_security_group" "web" {
  name = "web-sg"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  # Implicit dependency - Terraform knows to create SG first
  vpc_security_group_ids = [aws_security_group.web.id]
}
```

### Explicit Dependencies

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  depends_on = [
    aws_iam_role_policy.example,
    aws_cloudwatch_log_group.example
  ]
}
```

---

## Complex Types

### Object Type

```hcl
variable "instance_config" {
  type = object({
    name          = string
    instance_type = string
    ami           = string
    tags          = map(string)
    enable_monitoring = bool
  })
  
  default = {
    name              = "web-server"
    instance_type     = "t2.micro"
    ami               = "ami-12345678"
    tags              = { Environment = "dev" }
    enable_monitoring = false
  }
}
```

### Tuple Type

```hcl
variable "mixed_list" {
  type    = tuple([string, number, bool])
  default = ["hello", 42, true]
}
```

### Complex Nesting

```hcl
variable "vpc_config" {
  type = object({
    cidr_block = string
    subnets = list(object({
      name = string
      cidr = string
      az   = string
      type = string  # "public" or "private"
    }))
    tags = map(string)
  })
  
  default = {
    cidr_block = "10.0.0.0/16"
    subnets = [
      {
        name = "public-1"
        cidr = "10.0.1.0/24"
        az   = "us-west-2a"
        type = "public"
      },
      {
        name = "private-1"
        cidr = "10.0.2.0/24"
        az   = "us-west-2a"
        type = "private"
      }
    ]
    tags = {
      Environment = "production"
    }
  }
}
```

---

## Exam Tips

1. **Functions** - Know common functions: `lookup`, `merge`, `length`, `format`, `join`, `split`
2. **Dynamic blocks** - Used to generate multiple nested blocks from collections
3. **Data sources** - Read-only queries for existing infrastructure
4. **Type constraints** - Understand `list`, `map`, `set`, `object`, `tuple`
5. **File functions** - `file()`, `templatefile()`, `filebase64()`
6. **Collection functions** - `flatten()`, `distinct()`, `element()`, `lookup()`

---

[⬅️ Previous: State Management](07-state-management.md) | [Next: Cloud & Enterprise ➡️](09-cloud-enterprise.md)
