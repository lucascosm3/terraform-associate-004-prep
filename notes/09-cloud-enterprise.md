# 09 - Terraform Cloud and Enterprise

> 📊 **Exam Weight:** 2%

---

## Overview

Terraform Cloud and Terraform Enterprise are HashiCorp's managed and self-hosted offerings that provide collaboration, governance, and automation features beyond the open-source CLI.

| Feature | Open Source | Cloud | Enterprise |
|---------|-------------|-------|------------|
| **Pricing** | Free | SaaS (per user) | Self-hosted (license) |
| **Remote State** | Manual setup | Built-in | Built-in |
| **Remote Execution** | No | Yes | Yes |
| **Team Management** | No | Yes | Yes |
| **Sentinel (Policy)** | No | Yes | Yes |
| **Cost Estimation** | No | Yes | Yes |
| **SSO** | No | Yes | Yes |
| **Audit Logging** | No | Yes | Yes |
| **On-Premises** | N/A | No | Yes |

---

## Terraform Cloud

### Core Features

#### 1. Remote State Management

```hcl
terraform {
  cloud {
    organization = "my-organization"
    
    workspaces {
      name = "production"
    }
  }
}
```

**Benefits:**
- Automatic state locking
- State encryption at rest
- State versioning
- No need to manage S3/DynamoDB

#### 2. Remote Execution Modes

| Mode | Description |
|------|-------------|
| **Remote** | Runs in Terraform Cloud (agents) |
| **Local** | Runs on your machine, uses Cloud for state |

**Configuration:**
```hcl
# Settings in Terraform Cloud UI or API
```

#### 3. VCS Integration

Terraform Cloud integrates with:
- GitHub
- GitLab
- Bitbucket
- Azure DevOps

**Features:**
- Automatic runs on PR
- Speculative plans
- Run triggers on merge

#### 4. Workspace Management

```
Organization
├── Workspaces
│   ├── production
│   ├── staging
│   └── development
├── Variables
├── Run History
└── Settings
```

### Key Capabilities

#### Cost Estimation

- Estimates cloud costs before applying
- Supports AWS, Azure, GCP
- Integrates with policy checks

#### Policy as Code (Sentinel)

```sentinel
# Example: Restrict instance types
import "tfplan"

allowed_instance_types = ["t2.micro", "t2.small", "t3.micro"]

main = rule {
  all tfplan.resources.aws_instance as _, instances {
    all instances as _, r {
      r.applied.instance_type in allowed_instance_types
    }
  }
}
```

**Policy Levels:**
- **Advisory:** Warns but allows
- **Soft Mandatory:** Can be overridden
- **Hard Mandatory:** Cannot be overridden

#### Run Triggers

- **Auto-apply:** Automatically apply successful plans
- **Manual apply:** Require confirmation
- **Speculative plans:** Preview on pull requests

### Terraform Cloud Architecture

```
┌─────────────────────────────────────┐
│         Terraform Cloud             │
│  ┌─────────────────────────────┐   │
│  │     Terraform Workers       │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐  │   │
│  │  │ Run │ │ Run │ │ Run │  │   │
│  │  └──┬──┘ └──┬──┘ └──┬──┘  │   │
│  └─────┼───────┼───────┼─────┘   │
│        │       │       │         │
│  ┌─────┴───────┴───────┴─────┐   │
│  │    State Storage (Enc)    │   │
│  └───────────────────────────┘   │
└───────────────────────────────────┘
```

---

## Terraform Enterprise

Terraform Enterprise is the self-hosted version of Terraform Cloud.

### Additional Features

| Feature | Description |
|---------|-------------|
| **On-Premises** | Deploy in your data center |
| **Private Networking** | No internet exposure |
| **Air-Gapped** | Works without internet |
| **Custom Runners** | Use your own compute |
| **Advanced Security** | Enhanced audit logging |
| **SSO Integration** | SAML, LDAP, OIDC |

### Deployment Options

- **Online:** Connected installation
- **Air-gapped:** Isolated installation
- **Active-Active:** High availability setup

---

## Comparing Editions

### When to Use What?

| Scenario | Recommendation |
|----------|----------------|
| Individual learning | Open Source |
| Small team (3-5) | Terraform Cloud (Free tier) |
| Growing team | Terraform Cloud (Standard) |
| Large organization | Terraform Cloud (Plus) |
| Strict compliance requirements | Terraform Enterprise |
| Data residency requirements | Terraform Enterprise |
| Air-gapped environment | Terraform Enterprise |

---

## Terraform Cloud Configuration

### Connecting to Workspaces

**Option 1: CLI-driven**
```bash
# Login to Terraform Cloud
terraform login

# Initialize with Cloud backend
terraform init
```

**Option 2: VCS-driven**
```hcl
# Configure in Terraform Cloud UI
# Connect to GitHub/GitLab repository
# Runs triggered automatically
```

### Workspace Variables

```hcl
# Set in Terraform Cloud UI
# Environment Variables (marked as sensitive)
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY

# Terraform Variables
instance_type = "t2.micro"
region = "us-west-2"
```

### API-driven Workflows

```bash
# Trigger run via API
curl \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @payload.json \
  https://app.terraform.io/api/v2/runs
```

---

## Security Features

### Access Control

| Level | Permissions |
|-------|-------------|
| **Organization** | Owner, Member |
| **Team** | Manage policies, workspaces |
| **Workspace** | Read, Plan, Write, Admin |

### Sensitive Data

- Variables can be marked as **sensitive**
- Sensitive values are masked in logs
- State encryption at rest

```hcl
# In Terraform Cloud UI
variable "db_password" {
  value     = "supersecret"
  sensitive = true
}
```

### Audit Logging

- Track all runs and changes
- Export logs for compliance
- View who changed what and when

---

## Run Stages

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Pending  │ → │ Planning │ → │  Policy  │ → │  Cost    │ → │ Applying │
│          │   │          │   │  Check   │   │  Est.    │   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
                                                                   │
                                                            ┌──────────┐
                                                            │ Applied  │
                                                            │          │
                                                            └──────────┘
```

1. **Pending:** Waiting for worker
2. **Planning:** Creating execution plan
3. **Policy Check:** Running Sentinel policies (if configured)
4. **Cost Estimation:** Estimating cloud costs
5. **Applying:** Making changes
6. **Applied:** Changes complete

---

## Private Registry

Terraform Cloud/Enterprise includes a private module registry.

```hcl
# Use private module
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.0.0"
}
```

**Benefits:**
- Share modules within organization
- Version control
- Access control

---

## Exam Tips

1. **Terraform Cloud is SaaS** - Enterprise is self-hosted
2. **Remote execution** - Available in Cloud/Enterprise only
3. **Sentinel policies** - Policy as code for governance
4. **State management** - Built-in, no manual setup needed
5. **Cost estimation** - Shows estimated cloud costs
6. **VCS integration** - Automatic runs on PRs
7. **Private registry** - Share modules within org

---

## Quick Comparison

| Feature | OSS | Cloud | Enterprise |
|---------|-----|-------|------------|
| Remote state | DIY | ✅ | ✅ |
| Remote runs | ❌ | ✅ | ✅ |
| Team mgmt | ❌ | ✅ | ✅ |
| Sentinel | ❌ | ✅ | ✅ |
| Cost estimation | ❌ | ✅ | ✅ |
| Self-hosted | N/A | ❌ | ✅ |

---

[⬅️ Previous: Configurations](08-configurations.md) | [Back to Main ➡️](../README.md)
