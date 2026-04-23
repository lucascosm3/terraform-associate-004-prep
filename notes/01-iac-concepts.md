# 01 - Domain 1: Infrastructure as Code (IaC) with Terraform

> Exam Weight: ~11% | Tested on Terraform 1.12

---

## Objectives

- [x] **1a:** Explain what Infrastructure as Code (IaC) is
- [x] **1b:** Describe the advantages of IaC patterns
- [x] **1c:** Explain how Terraform manages multi-cloud, hybrid cloud, and service-agnostic workflows

---

## 1a. What is Infrastructure as Code (IaC)?

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure through machine-readable definition files rather than manual processes or interactive configuration tools.

### Core Principle

Instead of clicking through a console to create resources, you write code that describes your infrastructure's desired state, and an IaC tool makes the API calls to bring reality in line with that code.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

### Declarative vs Imperative IaC

| Aspect | Declarative | Imperative |
|--------|------------|------------|
| **Focus** | What the end state should be | How to reach the end state (step-by-step) |
| **Approach** | Define desired state; tool figures out how | Define specific commands to execute |
| **Idempotency** | Inherently idempotent | May not be idempotent |
| **State Tracking** | Maintains state file | Often stateless |
| **Examples** | Terraform, CloudFormation, Azure Bicep | Ansible, Chef, Puppet, Bash scripts |

```hcl
# Declarative (Terraform) - describe WHAT you want
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

```yaml
# Imperative (Ansible) - describe HOW to do it
- name: Create EC2 instance
  ec2:
    image: ami-0c55b159cbfafe1f0
    instance_type: t2.micro
    state: present
```

**Terraform is declarative** — you define the desired end state and Terraform determines the actions needed to reach it.

### Mutable vs Immutable Infrastructure

| Aspect | Mutable | Immutable |
|--------|---------|-----------|
| **Definition** | Servers are updated in-place | Servers are replaced entirely |
| **Updates** | In-place patches and upgrades | Deploy new version; destroy old |
| **Consistency** | Risk of configuration drift | Guaranteed consistency |
| **Rollback** | Difficult; need to reverse changes | Simple; redeploy previous version |
| **Tools** | Ansible, Chef, Puppet | Terraform, Packer, Docker, Kubernetes |

**Terraform follows the immutable approach** for most resources. When a resource attribute that cannot be updated in-place changes, Terraform destroys the old resource and creates a new one.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-new"      # Changing AMI triggers replacement
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true  # Create new before destroying old
  }
}
```

### Idempotency

Idempotency means that applying the same operation multiple times produces the same result. Running `terraform apply` on already-applied configuration results in no changes.

```bash
terraform apply   # Creates infrastructure
terraform apply   # No changes (already exists as defined)
```

This is a key advantage of declarative IaC — you can safely re-apply configuration without fear of duplication or side effects.

---

## 1b. Advantages of IaC Patterns

| Advantage | Description |
|-----------|-------------|
| **Version Control** | Track changes, rollback, full audit history via Git |
| **Consistency** | Same environment every time; eliminates "snowflake" servers |
| **Automation** | Reduce human error; faster provisioning; CI/CD integration |
| **Collaboration** | Teams review, share, and reuse infrastructure code |
| **Documentation** | Code serves as living documentation of infrastructure |
| **Scalability** | Easily replicate infrastructure across environments |
| **Cost Transparency** | See exactly what resources are defined and their relationships |
| **Disaster Recovery** | Rebuild entire infrastructure from code in minutes |

### IaC Workflow Benefits

```
Code → Review → Test → Deploy → Monitor
  ↑                                  |
  └────────── Feedback ──────────────┘
```

1. **Reproducibility** — Create identical environments (dev, staging, prod)
2. **Transparency** — All changes are tracked in version control
3. **Validation** — Run tests and policy checks before deployment
4. **Self-documentation** — The code IS the documentation

---

## 1c. Multi-Cloud, Hybrid Cloud, and Service-Agnostic Workflows

### Multi-Cloud

Terraform's provider-based architecture allows you to manage infrastructure across multiple cloud platforms using a single tool and a consistent workflow.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
}

provider "google" {
  project = "my-project"
  region  = "us-central1"
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "azurerm_linux_virtual_machine" "app" {
  name                = "app-vm"
  resource_group_name  = azurerm_resource_group.main.name
  location            = "East US"
  size                = "Standard_B1s"
  # ...
}
```

### Provider Aliases for Multi-Region

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

resource "aws_instance" "east_app" {
  provider      = aws
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "aws_instance" "west_app" {
  provider      = aws.west
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

### Hybrid Cloud

Terraform can manage resources that span on-premises and cloud environments simultaneously. This is critical for hybrid-cloud architectures.

```hcl
# On-premises VMware resources
provider "vsphere" {
  user           = "administrator@vsphere.local"
  password       = var.vsphere_password
  vsphere_server = "vcenter.local"
}

resource "vsphere_virtual_machine" "onprem" {
  name             = "onprem-app"
  resource_pool_id = data.vsphere_resource_pool.pool.id
  datastore_id     = data.vsphere_datastore.datastore.id
  # ...
}

# Cloud resources
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "cloud" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# VPN connecting on-prem to cloud
resource "aws_vpn_gateway" "vpn" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "onprem-vpn"
  }
}
```

### Service-Agnostic Workflows

Terraform is not limited to cloud providers. It can manage SaaS platforms, CI/CD tools, monitoring, DNS, and more — all with the same declarative workflow.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
    github = {
      source  = "integrations/github"
    }
    datadog = {
      source  = "DataDog/datadog"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
    }
  }
}

resource "github_repository" "app" {
  name        = "my-application"
  description = "Application repository"
  visibility  = "private"
}

resource "datadog_monitor" "high_cpu" {
  name   = "High CPU"
  type   = "metric alert"
  message = "CPU is high on {{host.name}}"
  query  = "avg(last_5m):avg:system.cpu.user{*} > 80"
}

resource "cloudflare_record" "app" {
  zone_id = var.cloudflare_zone_id
  name    = "app"
  value   = aws_lb.app.dns_name
  type    = "CNAME"
  proxied = true
}
```

**Key Takeaway:** Terraform's provider plugin model means the same Write → Plan → Apply workflow works identically whether you're provisioning AWS VPCs, GitHub repositories, Datadog monitors, or Cloudflare DNS records.

### Terraform vs Other IaC Tools

| Feature | Terraform | CloudFormation | Ansible | Pulumi |
|----------|-----------|----------------|---------|--------|
| **Approach** | Declarative | Declarative | Procedural + Declarative | Imperative/Declarative |
| **Multi-cloud** | Yes (3000+ providers) | No (AWS only) | Yes | Yes |
| **State Management** | Yes (local/remote) | Yes (AWS managed) | No | Yes |
| **Language** | HCL | JSON/YAML | YAML | Python/TypeScript/Go |
| **Agentless** | Yes | Yes (managed by AWS) | Yes (SSH/WinRM) | Yes |
| **Primary Use** | Provisioning | Provisioning (AWS) | Configuration | Provisioning |

---

## Exam Tips

1. **Terraform is declarative** — you describe the desired state, Terraform figures out how to reach it
2. **Terraform is immutable** — resources are typically replaced, not modified in-place
3. **Idempotency** — running `terraform apply` multiple times produces the same result
4. **Multi-cloud is a core differentiator** — same tool, same workflow, many providers
5. **Provider-based architecture** — Terraform manages any API, not just cloud infrastructure
6. **IaC advantages** — version control, consistency, automation, collaboration, documentation
7. **Terraform vs Ansible** — Terraform provisions, Ansible configures; they complement each other

---

## Quick Review Questions

1. **Q:** What is the main difference between declarative and imperative IaC?
   **A:** Declarative defines the desired end state; imperative defines the steps to reach it.

2. **Q:** Why is idempotency important in IaC?
   **A:** Applying the same configuration multiple times yields the same result, ensuring consistency and safety.

3. **Q:** How does Terraform handle multi-cloud environments?
   **A:** Through provider plugins — each provider wraps a specific API, and you use the same Write → Plan → Apply workflow across all of them.

4. **Q:** What is the difference between mutable and immutable infrastructure?
   **A:** Mutable infrastructure is updated in-place (risk of drift); immutable infrastructure replaces resources entirely (guaranteed consistency).

5. **Q:** Can Terraform manage non-cloud resources?
   **A:** Yes — Terraform providers exist for SaaS platforms (GitHub, Datadog), DNS (Cloudflare), CI/CD, and more.

---

[Next: Terraform Fundamentals ➡️](02-terraform-fundamentals.md)