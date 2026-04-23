# 02 - Terraform's Purpose

> 📊 **Exam Weight:** 9%

---

## Why Terraform?

Terraform is an open-source **infrastructure as code** software tool created by HashiCorp. It enables users to define and provision data center infrastructure using a declarative configuration language.

### Key Characteristics

| Feature | Description |
|---------|-------------|
| **Multi-cloud** | Works with AWS, Azure, GCP, and 100+ providers |
| **Platform Agnostic** | Not tied to a single cloud vendor |
| **State Management** | Tracks infrastructure state for accurate updates |
| **Execution Plans** | Preview changes before applying |
| **Resource Graph** | Parallelizes creation/modification where possible |
| **Open Source** | Free, with enterprise options available |

---

## Terraform vs Other IaC Tools

### Terraform vs CloudFormation

| Aspect | Terraform | CloudFormation |
|--------|-----------|----------------|
| **Vendor Lock-in** | None (multi-cloud) | AWS only |
| **Language** | HCL (Human-readable) | JSON/YAML |
| **State** | Self-managed (local/remote) | AWS-managed |
| **Drift Detection** | Built-in (`terraform plan`) | AWS Config/CloudFormation drift |
| **Modularity** | Native modules | Nested stacks |

### Terraform vs Ansible

| Aspect | Terraform | Ansible |
|--------|-----------|---------|
| **Primary Focus** | Infrastructure provisioning | Configuration management |
| **Approach** | Declarative | Procedural + Declarative |
| **State** | Maintains state file | Stateless |
| **Best For** | Creating resources | Configuring servers |
| **Together** | Terraform provisions, Ansible configures |

### Terraform vs Pulumi

| Aspect | Terraform | Pulumi |
|--------|-----------|--------|
| **Language** | HCL | Python, TypeScript, Go, C# |
| **Learning Curve** | Lower (DSL) | Higher (requires programming) |
| **State** | Local/remote backends | Similar backend options |
| **Use Case** | Infrastructure teams | Developer-centric teams |

---

## Terraform Core Concepts

### 1. Providers

Providers are plugins that enable Terraform to interact with cloud providers, SaaS providers, and other APIs.

```hcl
# AWS Provider
provider "aws" {
  region = "us-west-2"
}

# Multiple providers (aliasing)
provider "aws" {
  alias  = "west"
  region = "us-west-1"
}

provider "aws" {
  alias  = "east"
  region = "us-east-1"
}
```

### 2. Resources

Resources are the most important element in Terraform - they represent infrastructure objects.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

### 3. Data Sources

Data sources allow Terraform to use information defined outside of Terraform or from other Terraform configurations.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  owners = ["099720109477"] # Canonical
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}
```

---

## Terraform Workflow

```
Write → Initialize → Plan → Apply → Destroy
```

1. **Write** configuration in `.tf` files
2. **Initialize** with `terraform init`
3. **Plan** changes with `terraform plan`
4. **Apply** changes with `terraform apply`
5. **Destroy** when done with `terraform destroy`

---

## Terraform Editions

### Terraform Open Source (CLI)
- Free, open-source
- Local state by default
- Self-managed

### Terraform Cloud
- SaaS offering by HashiCorp
- Remote state management
- Team collaboration features
- Policy as code (Sentinel)
- Cost estimation

### Terraform Enterprise
- Self-hosted version of Terraform Cloud
- On-premises deployment
- Advanced security and compliance features

---

## When to Use Terraform

### ✅ Best For
- Multi-cloud infrastructure
- Cloud resource provisioning
- Immutable infrastructure
- Version-controlled infrastructure
- Team collaboration on infrastructure

### ❌ Not Ideal For
- Application deployment (use CI/CD tools)
- Server configuration (use Ansible/Chef)
- Container orchestration (use Kubernetes)

---

## Exam Tips

1. **Terraform is NOT tied to a single cloud provider** - this is a key differentiator
2. **Terraform maintains state** - this enables accurate change tracking
3. **Terraform Cloud is SaaS** - Enterprise is self-hosted
4. **Terraform is for provisioning** - not configuration management
5. **Terraform uses HCL** - HashiCorp Configuration Language

---

## Quick Review Questions

1. **Q:** What is Terraform's primary advantage over CloudFormation?
   **A:** Multi-cloud support (not tied to a single vendor).

2. **Q:** Can Terraform and Ansible be used together?
   **A:** Yes! Terraform for provisioning infrastructure, Ansible for configuring it.

3. **Q:** What is the difference between Terraform Cloud and Enterprise?
   **A:** Cloud is SaaS (HashiCorp-hosted), Enterprise is self-hosted.

4. **Q:** What language does Terraform use?
   **A:** HCL (HashiCorp Configuration Language).

---

[⬅️ Previous: IaC Concepts](01-infrastructure-as-code.md) | [Next: Terraform Basics ➡️](03-terraform-basics.md)
