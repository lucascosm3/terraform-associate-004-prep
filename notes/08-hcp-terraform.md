# 08 - Domain 8: HCP Terraform

> Exam Weight: ~5% | Tested on Terraform 1.12

---

## Objectives

- [x] **8a:** Use HCP Terraform to create infrastructure (workspaces, remote operations)
- [x] **8b:** Describe HCP Terraform collaboration and governance features (Explorer, Private Registry, Change requests, Policy enforcement, Projects, Health, Teams)
- [x] **8c:** Describe how to organize and use HCP Terraform workspaces and projects (Run triggers, Variable sets, Projects)
- [x] **8d:** Configure and use HCP Terraform integration (CLI-driven workflow, cloud block, migrate state, dynamic provider credentials)

---

## 8a. Use HCP Terraform to Create Infrastructure

### What Is HCP Terraform?

HCP Terraform (formerly "Terraform Cloud") is HashiCorp's managed SaaS platform for running Terraform operations in a consistent, collaborative environment.

| Feature | Open Source CLI | HCP Terraform |
|---------|----------------|---------------|
| **State Storage** | Local or manual remote | Built-in remote state |
| **Remote Execution** | No | Yes |
| **Team Collaboration** | Manual | Built-in |
| **Policy Enforcement** | No | Sentinel policies |
| **Cost Estimation** | No | Available |
| **VCS Integration** | Manual | Built-in |
| **Private Registry** | No | Available |
| **Variable Sets** | No | Available |
| **SSO/SAML** | No | Available |

### Workspaces

A workspace is the fundamental unit in HCP Terraform — it contains a Terraform configuration, variables, state, and run history.

```
Organization
├── Workspaces
│   ├── production-app
│   ├── staging-app
│   ├── dev-app
│   └── networking
├── Variable Sets
├── Projects
└── Settings
```

**Workspace types:**
- **VCS-driven** — connected to a Git repository; runs triggered by commits/PRs
- **CLI-driven** — runs triggered from local `terraform` CLI
- **API-driven** — runs triggered via API calls (automation)

### CLI-Driven Workflow

The CLI-driven workflow lets you use the familiar `terraform` CLI while HCP Terraform handles remote execution and state.

```bash
# Login to HCP Terraform
terraform login

# Initialize with HCP Terraform backend
terraform init
```

```hcl
# Configure HCP Terraform backend
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}
```

**How it works:**
1. You write configuration locally
2. `terraform init` connects to HCP Terraform
3. `terraform plan/apply` runs on HCP Terraform infrastructure
4. State is stored remotely in HCP Terraform
5. Results are streamed back to your terminal

### VCS-Driven Workflow

Connect a workspace to a Git repository for automatic runs:

1. Connect to GitHub, GitLab, or Bitbucket
2. Configure triggers (push to main, pull requests)
3. HCP Terraform runs `terraform plan` on PRs
4. HCP Terraform runs `terraform apply` on merge to main

**VCS Integration Settings:**
- **Branch** — which branch to monitor
- **Workspace directory** — subdirectory containing Terraform configs
- **Trigger prefix** — only trigger on changes to specific directories
- **Auto-apply** — automatically apply successful plans

### Remote Operations

HCP Terraform executes Terraform operations on its infrastructure:

```
Local CLI                        HCP Terraform
   │                                  │
   ├── terraform init ──────────────>│── Downloads providers
   │                                  │── Stores in workspace
   │                                  │
   ├── terraform plan ──────────────>│── Runs plan remotely
   │<── Plan output ─────────────────│── Streams results back
   │                                  │
   ├── terraform apply ─────────────>│── Locks workspace
   │<── Apply output ────────────────│── Runs apply remotely
   │                                  │── Updates remote state
   │                                  │── Releases lock
```

**Benefits of remote execution:**
- Consistent execution environment
- Secure credential storage
- Audit trail of all operations
- No need for local cloud credentials

---

## 8b. HCP Terraform Collaboration and Governance Features

### HCP Terraform Explorer

Explorer provides visibility into your infrastructure across all workspaces:

- Search and filter resources across workspaces
- View resource inventory by type, provider, and workspace
- Identify drift and cost trends
- Answer questions like "How many EC2 instances are running?"

### Private Registry

HCP Terraform includes a private registry for sharing modules and providers within your organization.

```hcl
# Use a private module from HCP Terraform
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.0.0"
}
```

**Private Registry Features:**
- Share modules within your organization
- Version control for internal modules
- Access control and permissions
- Support for private providers
- Verified module badges

**Publishing to Private Registry:**
1. Create a Git repository with a valid Terraform module
2. Connect the repository to HCP Terraform
3. Tag a release to publish a version
4. The module becomes available at `app.terraform.io/<org>/<name>/<provider>`

### Change Requests

Change Requests allow team members to propose infrastructure changes through HCP Terraform's UI:

- Propose changes to workspace variables or configuration
- Review and approve changes before they are applied
- Audit trail of all proposed and applied changes
- Integration with existing approval workflows

### Policy Enforcement (Sentinel)

Sentinel is HashiCorp's policy-as-code framework integrated with HCP Terraform:

```sentinel
# Example: Restrict instance types
import "tfplan"

allowed_instance_types = ["t2.micro", "t2.small", "t3.micro", "t3.small"]

main = rule {
  all tfplan.resources.aws_instance as _, instances {
    all instances as _, r {
      r.applied.instance_type in allowed_instance_types
    }
  }
}
```

**Policy enforcement levels:**
- **Advisory** — Warns but allows the run to continue
- **Soft mandatory** — Can be overridden by organization admins
- **Hard mandatory** — Cannot be overridden; the run fails

**When policies are evaluated:**
- After `terraform plan`
- Before `terraform apply`
- Policies can block the apply if they fail at hard mandatory level

### Projects

Projects group related workspaces for organization and governance:

```
Organization
├── Project: Networking
│   ├── Workspace: vpc-production
│   ├── Workspace: vpc-staging
│   └── Workspace: vpc-dev
├── Project: Application
│   ├── Workspace: app-production
│   ├── Workspace: app-staging
│   └── Workspace: app-dev
└── Project: Database
    ├── Workspace: db-production
    └── Workspace: db-staging
```

**Project features:**
- Group related workspaces
- Apply variable sets to all workspaces in a project
- Manage permissions at the project level
- Organize infrastructure by team or service

### Health Assessments

HCP Terraform performs health assessments on workspaces:

- **Drift detection** — identifies when infrastructure has changed outside of Terraform
- **Cost estimation** — estimates cloud costs before applying changes
- **Policy compliance** — checks against Sentinel policies
- **Resource health** — monitors resource status

### Teams and Permissions

HCP Terraform provides fine-grained access control:

| Level | Permissions |
|-------|-------------|
| **Organization** | Manage organization settings, billing, users |
| **Project** | Manage workspaces within a project |
| **Team** | Group users with shared permissions |
| **Workspace** | Read, Plan, Write, Admin |

**Team features:**
- Assign permissions at organization and workspace levels
- SSO integration (SAML, OIDC)
- API token management
- Two-factor authentication

---

## 8c. Organize HCP Terraform Workspaces and Projects

### Run Triggers

Run triggers automatically start a run in one workspace when another workspace completes successfully.

```hcl
# Example: Network workspace triggers Application workspace
# When vpc-production workspace applies successfully,
# it triggers a run on app-production workspace
```

**Configuration in HCP Terraform UI:**
1. Go to workspace Settings → Run Triggers
2. Select the source workspace
3. The destination workspace will automatically run when the source succeeds

**Common use cases:**
- VPC changes trigger application redeployment
- Database changes trigger application updates
- Base infrastructure changes trigger dependent workspace runs

### Variable Sets

Variable sets allow you to define variables that are shared across multiple workspaces.

**Creating a Variable Set:**
1. Navigate to Organization Settings → Variable Sets
2. Create a new set (e.g., "Production Common")
3. Add variables (e.g., `environment = "production"`)
4. Apply the set to specific workspaces or projects

**Types of variable sets:**
- **Organization-wide** — applied to all workspaces
- **Project-scoped** — applied to all workspaces in a project
- **Workspace-specific** — applied to selected workspaces

**Variable priority (highest to lowest):**
1. Workspace-specific variables
2. Variable sets (applied to workspace)
3. `.tfvars` files
4. Default values in variable declarations

```hcl
# Common variables in a variable set
# Variable Set: "AWS Credentials"
AWS_ACCESS_KEY_ID     = "AKIAIOSFODNN7EXAMPLE"  # (sensitive)
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/..."      # (sensitive)

# Variable Set: "Production Environment"
environment = "production"
region      = "us-east-1"

# Variable Set: "Staging Environment"
environment = "staging"
region      = "us-west-2"
```

### Workspace Organization Patterns

**Pattern 1: Environment per Workspace**
```
Organization
├── Project: Infrastructure
│   ├── Workspace: vpc-production
│   ├── Workspace: vpc-staging
│   └── Workspace: vpc-dev
```

**Pattern 2: Component per Workspace**
```
Organization
├── Project: Production
│   ├── Workspace: networking-production
│   ├── Workspace: database-production
│   └── Workspace: application-production
```

**Pattern 3: Single Workspace with Workspaces (Terraform CLI `workspace`)**
```hcl
# Use Terraform CLI workspaces for environments
terraform workspace new production
terraform workspace new staging
terraform workspace new dev
```

### Projects for Organization

```hcl
# Create a project in HCP Terraform
# Projects group related workspaces and enforce governance

# Project: Platform
# ├── Variable Set: "Platform Common" (applies to all workspaces)
# │   ├── region = "us-east-1"
# │   └── environment = "production"
# ├── Workspace: networking
# ├── Workspace: database
# └── Workspace: application
```

---

## 8d. Configure and Use HCP Terraform Integration

### CLI-Driven Workflow Setup

**Step 1: Login to HCP Terraform**

```bash
# Authenticate with HCP Terraform
terraform login

# This opens a browser to generate an API token
# Token is stored in ~/.terraform.d/credentials.tf1.json
```

**Step 2: Configure the Cloud Block**

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}
```

**Step 3: Initialize and Run**

```bash
terraform init    # Connects to HCP Terraform
terraform plan     # Runs plan remotely
terraform apply    # Runs apply remotely
```

### Cloud Block Configuration

```hcl
# Single workspace
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}

# Workspace tags (for dynamic workspace selection)
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      tags = ["production", "networking"]
    }
  }
}
```

### Migrating State to HCP Terraform

**From local state:**

```bash
# Step 1: Add the cloud block to your configuration
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-workspace"
    }
  }
}

# Step 2: Initialize and migrate state
terraform init -migrate-state

# Step 3: Verify state migrated successfully
terraform state list
```

**From S3 backend:**

```bash
# Step 1: Replace the S3 backend block with cloud block
# OLD:
# terraform {
#   backend "s3" {
#     bucket = "my-terraform-state"
#     key    = "prod/terraform.tfstate"
#     region = "us-east-1"
#   }
# }

# NEW:
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}

# Step 2: Initialize and migrate
terraform init -migrate-state
```

**From any backend without migration:**

```bash
# Disconnect from old backend without migrating state
terraform init -reconfigure
```

### Dynamic Provider Credentials

HCP Terraform can dynamically generate short-lived cloud provider credentials for runs, eliminating the need to store long-lived credentials as workspace variables.

```hcl
# Dynamic AWS credentials via OIDC
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# No need to set AWS_ACCESS_KEY_ID/AWS_SECRET_ACCESS_KEY!
# HCP Terraform generates temporary credentials via OIDC
```

**Benefits of dynamic provider credentials:**
- No long-lived credentials stored in HCP Terraform
- Short-lived tokens that automatically rotate
- OIDC-based trust relationships with cloud providers
- Reduced blast radius if credentials are compromised

**Supported providers:**
- AWS (via OIDC federation)
- Azure (via OIDC federation)
- GCP (via Workload Identity Federation)

**Configuration in HCP Terraform:**
1. Create a trust relationship between HCP Terraform and your cloud provider
2. Configure the dynamic credentials in workspace settings
3. HCP Terraform automatically generates temporary credentials for each run

### API Token Authentication

```bash
# Generate an API token in HCP Terraform UI
# Store as environment variable
export TFE_TOKEN="your-api-token"

# Or use the credentials file
# ~/.terraform.d/credentials.tf1.json
```

### Environment Variables for HCP Terraform

```bash
# API token
export TFE_TOKEN="your-api-token"

# HCP Terraform address (for Enterprise)
export TFE_ADDRESS="https://app.terraform.io"

# Skip verification (not recommended for production)
export TF_SKIP_VERIFY="1"
```

### Workspace Variables

In HCP Terraform, variables can be set at the workspace level:

| Variable Type | Prefix | Description |
|---------------|--------|-------------|
| **Terraform variable** | `TF_VAR_<name>` | Standard Terraform input variable |
| **Environment variable** | No prefix | Shell environment variable (e.g., `AWS_REGION`) |
| **Sensitive** | Marked per variable | Hidden in UI and logs |

**Setting variables in the workspace:**
1. Navigate to workspace → Variables
2. Add Terraform variables or environment variables
3. Mark sensitive variables

### Sentinel Policy Enforcement

```sentinel
# Policy: Require specific tags on all resources
import "tfplan"

main = rule {
  all tfplan.resources as _, r {
    "Environment" in r.applied.tags
  }
}
```

**Policy evaluation flow:**
1. `terraform plan` runs in HCP Terraform
2. Sentinel policies evaluate the plan
3. If policies pass (or are advisory), `terraform apply` proceeds
4. If hard mandatory policies fail, the run is blocked

### Cost Estimation

HCP Terraform provides cost estimation for planned changes:

- Estimates cost before and after applying changes
- Shows monthly cost delta
- Supports AWS, Azure, and GCP pricing
- Available in the plan output

### HCP Terraform vs Terraform Enterprise

| Feature | HCP Terraform | Terraform Enterprise |
|---------|---------------|---------------------|
| **Deployment** | SaaS (HashiCorp-hosted) | Self-hosted |
| **State Storage** | HashiCorp-managed | Self-managed |
| **On-Premises** | No | Yes |
| **Air-Gapped** | No | Yes |
| **Private Networking** | No | Yes |
| **SSO** | Yes | Yes |
| **Audit Logging** | Yes | Enhanced |
| **Custom Runners** | No | Yes |
| **Dynamic Provider Creds** | Yes | Yes |

---

## Exam Tips

1. **HCP Terraform** (not "Terraform Cloud") — use the new name
2. **Three workflow types** — VCS-driven, CLI-driven, API-driven
3. **Workspaces** are the fundamental unit — contain config, variables, state, runs
4. **Variable sets** — shared variables across multiple workspaces
5. **Run triggers** — automatically start a workspace when another workspace succeeds
6. **Projects** — group workspaces for organization and governance
7. **Private Registry** — share modules and providers within your organization
8. **Sentinel** — policy-as-code; three levels: advisory, soft mandatory, hard mandatory
9. **Dynamic provider credentials** — OIDC-based short-lived credentials (no stored secrets)
10. **Migrate state** — `terraform init -migrate-state` to move to HCP Terraform
11. **Explorer** — visibility into resources across workspaces
12. **Health assessments** — drift detection, cost estimation, policy compliance

---

## Quick Review Questions

1. **Q:** What are the three types of Terraform workflows in HCP Terraform?
   **A:** VCS-driven (triggered by Git), CLI-driven (triggered from local CLI), and API-driven (triggered via API).

2. **Q:** What is a Variable Set?
   **A:** A set of variables shared across multiple workspaces. They can be organization-wide, project-scoped, or workspace-specific.

3. **Q:** What are Run Triggers?
   **A:** They automatically start a run in one workspace when another workspace completes successfully.

4. **Q:** What is the Private Registry?
   **A:** A feature for sharing Terraform modules and providers within an organization on HCP Terraform.

5. **Q:** What are the three Sentinel policy enforcement levels?
   **A:** Advisory (warn only), Soft Mandatory (can be overridden by admins), Hard Mandatory (cannot be overridden).

6. **Q:** How do you migrate state to HCP Terraform?
   **A:** Add the `cloud` block to configuration and run `terraform init -migrate-state`.

7. **Q:** What are Dynamic Provider Credentials?
   **A:** HCP Terraform generates short-lived OIDC-based credentials for cloud providers, eliminating the need for stored long-lived credentials.

8. **Q:** What is HCP Terraform Explorer?
   **A:** A feature that provides visibility into resources across all workspaces, allowing search, filtering, and drift detection.

---

[⬅️ Previous: Maintain Infrastructure](07-maintain-infrastructure.md) | [Back to Main ↩️](../README.md)