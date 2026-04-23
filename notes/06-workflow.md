# 06 - Terraform Workflow

> 📊 **Exam Weight:** 9%

---

## The Core Workflow

Terraform follows a standard workflow that applies to both individual developers and teams:

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│   Write     │ -> │  Initialize  │ -> │    Plan     │ -> │    Apply    │
│  (Config)   │    │    (Init)    │    │   (Plan)    │    │   (Apply)   │
└─────────────┘    └──────────────┘    └─────────────┘    └─────────────┘
                                                                 │
                                                                 v
                                                          ┌─────────────┐
                                                          │   Destroy   │
                                                          │  (Destroy)  │
                                                          └─────────────┘
```

---

## Step 1: Write

Create or modify Terraform configuration files.

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

**Best Practices:**
- Use version control (Git)
- Run `terraform fmt` before committing
- Validate with `terraform validate`

---

## Step 2: Initialize

Prepare the working directory for use with Terraform.

```bash
terraform init
```

**What happens:**
- Downloads required providers
- Sets up backend configuration
- Downloads modules
- Creates `.terraform/` directory

```bash
# Initialize with upgrade
terraform init -upgrade

# Initialize with backend migration
terraform init -migrate-state
```

---

## Step 3: Plan

Preview the changes Terraform will make.

```bash
terraform plan
```

**What happens:**
- Reads current state
- Compares with configuration
- Proposes changes
- Shows execution plan

```bash
# Save plan to file
terraform plan -out=tfplan

# Plan with variables
terraform plan -var="instance_type=t2.large"

# Detailed exit codes
terraform plan -detailed-exitcode
```

### Plan Output Example

```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                          = "ami-0c55b159cbfafe1f0"
      + instance_type                = "t2.micro"
      + tags                         = {
          + "Name" = "WebServer"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

---

## Step 4: Apply

Execute the planned changes.

```bash
terraform apply
```

**What happens:**
- Shows plan (if not already shown)
- Asks for confirmation (unless `-auto-approve`)
- Creates/updates/deletes resources
- Updates state file

```bash
# Apply without confirmation
terraform apply -auto-approve

# Apply saved plan
terraform apply tfplan

# Apply specific target
terraform apply -target=aws_instance.web
```

### Apply Confirmation

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]
aws_instance.web: Creation complete after 15s [id=i-1234567890abcdef0]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

---

## Step 5: Destroy (When Needed)

Remove all resources managed by Terraform.

```bash
terraform destroy
```

**What happens:**
- Shows plan of destruction
- Asks for confirmation
- Destroys all resources
- Updates state file

```bash
# Destroy without confirmation
terraform destroy -auto-approve

# Destroy specific target
terraform destroy -target=aws_instance.web
```

**⚠️ Warning:** This is irreversible! Always double-check.

---

## Workflow Variations

### Individual Development

```bash
# Daily workflow
git pull                    # Get latest
terraform init              # Initialize
terraform validate          # Validate
terraform plan              # Review changes
terraform apply             # Apply changes
git add . && git commit     # Commit
git push                    # Push
```

### Team Workflow

```bash
# Feature branch workflow
git checkout -b feature/new-resource
# ... make changes ...
terraform fmt
terraform validate
terraform plan
git add . && git commit -m "feat: add new resource"
git push origin feature/new-resource
# Create Pull Request
# Peer review
# Merge to main
```

### CI/CD Workflow

```yaml
# GitHub Actions example
steps:
  - uses: actions/checkout@v3
  
  - name: Setup Terraform
    uses: hashicorp/setup-terraform@v2
    
  - name: Terraform Format
    run: terraform fmt -check
    
  - name: Terraform Init
    run: terraform init
    
  - name: Terraform Validate
    run: terraform validate
    
  - name: Terraform Plan
    run: terraform plan -out=tfplan
    
  - name: Terraform Apply
    if: github.ref == 'refs/heads/main'
    run: terraform apply -auto-approve tfplan
```

---

## Workflow Best Practices

### 1. Always Review Plans

Never apply blindly - always review the plan output:

```bash
terraform plan -out=tfplan
# Review tfplan file
cat tfplan | terraform show -  # Show plan details
terraform apply tfplan
```

### 2. Use Version Control

```bash
# Track everything except:
# - .terraform/
# - *.tfstate
# - *.tfstate.*
# - *.tfvars (if contains secrets)

echo '.terraform/' >> .gitignore
echo '*.tfstate*' >> .gitignore
```

### 3. State Management

- Never manually edit state files
- Use remote state for teams
- Enable state locking

### 4. Resource Targeting

Use sparingly to avoid dependency issues:

```bash
# Only when necessary
terraform apply -target=aws_instance.web
```

### 5. Validation Pipeline

Always validate before applying:

```bash
terraform fmt -check
terraform validate
terraform plan
```

---

## Common Workflow Commands

```bash
# Complete workflow
terraform init && terraform validate && terraform plan && terraform apply

# With auto-approve
terraform init && terraform apply -auto-approve

# Format and validate
terraform fmt && terraform validate

# Refresh state (sync with reality)
terraform refresh  # Or: terraform apply -refresh-only

# Check drift
terraform plan -detailed-exitcode
# Exit 0 = no drift
# Exit 2 = drift detected
```

---

## Troubleshooting Workflow Issues

### State Lock Errors

```bash
# If state is locked
terraform force-unlock <LOCK_ID>
```

### Init Errors

```bash
# Reinitialize
rm -rf .terraform/
terraform init
```

### Provider Issues

```bash
# Upgrade providers
terraform init -upgrade
```

---

## Exam Tips

1. **Standard workflow order** - Write → Init → Plan → Apply → (Destroy)
2. **Always review plans** - Before applying any changes
3. **State locking** - Prevents concurrent modifications
4. **Refresh vs Apply** - Refresh only updates state; Apply makes changes
5. **Destroy caution** - Irreversible; destroys all managed resources

---

[⬅️ Previous: Modules](05-modules.md) | [Next: State Management ➡️](07-state-management.md)
