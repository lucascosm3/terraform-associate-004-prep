# 01 - Infrastructure as Code (IaC) Concepts

> 📊 **Exam Weight:** 11%

---

## What is Infrastructure as Code (IaC)?

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure through code files rather than manual processes.

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Version Control** | Track changes, rollback, audit history |
| **Consistency** | Same environment every time, no drift |
| **Automation** | Reduce human error, faster provisioning |
| **Collaboration** | Teams can review and share infrastructure code |
| **Documentation** | Code serves as living documentation |
| **Scalability** | Easily replicate infrastructure across environments |

---

## Types of IaC Approaches

### 1. Declarative (Functional)

**Focus:** What the end state should be

- Define the desired state
- Tool determines how to achieve it
- Idempotent (same result every time)

**Examples:** Terraform, CloudFormation, ARM Templates, Azure Bicep

```hcl
# Terraform - Declarative Example
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
# Terraform ensures the instance exists with these properties
```

### 2. Imperative (Procedural)

**Focus:** How to achieve the state (step-by-step)

- Define specific commands to execute
- Order matters
- May not be idempotent

**Examples:** Ansible, Chef, Puppet (procedural aspects), Bash scripts

```yaml
# Ansible - Imperative Example (partially declarative too)
- name: Install nginx
  apt:
    name: nginx
    state: present
    
- name: Start nginx service
  service:
    name: nginx
    state: started
```

---

## Mutable vs Immutable Infrastructure

| Aspect | Mutable | Immutable |
|--------|---------|-----------|
| **Definition** | Can be modified after creation | Cannot be changed; replaced entirely |
| **Updates** | In-place changes | Deploy new version, destroy old |
| **Consistency** | Risk of configuration drift | Guaranteed consistency |
| **Rollback** | Complex | Simple (deploy previous version) |
| **Tools** | Ansible, Chef, Puppet | Terraform, Packer, Docker, Kubernetes |

**Terraform follows the immutable approach:** When you change a resource, Terraform destroys the old one and creates a new one (depending on the resource type).

---

## Key IaC Concepts

### Idempotency

The property that applying the same operation multiple times produces the same result.

```bash
# Terraform is idempotent
terraform apply  # Creates infrastructure
terraform apply  # No changes (already exists)
```

### State Management

IaC tools track the current state of infrastructure to:
- Determine what changes are needed
- Map configuration to real resources
- Enable collaboration

### Provider Abstraction

IaC tools abstract cloud provider APIs:
```hcl
# Same syntax works across providers
resource "aws_instance" "web" { }      # AWS
resource "azurerm_linux_vm" "web" { }  # Azure  
resource "google_compute_instance" "web" { }  # GCP
```

---

## Terraform vs Other IaC Tools

| Feature | Terraform | CloudFormation | Ansible | Pulumi |
|---------|-----------|----------------|---------|--------|
| **Approach** | Declarative | Declarative | Procedural | Imperative/Declarative |
| **Multi-cloud** | Yes | No (AWS only) | Yes | Yes |
| **State Management** | Yes (local/remote) | Yes (AWS managed) | No | Yes |
| **Language** | HCL | JSON/YAML | YAML | Python/TypeScript/Go |
| **Agentless** | Yes | Yes (managed by AWS) | Yes (SSH/WinRM) | Yes |

---

## Exam Tips

1. **Know the difference** between declarative and imperative approaches
2. **Understand idempotency** and why it's important
3. **Remember Terraform is declarative** - you define "what", not "how"
4. **Terraform is immutable** - resources are replaced, not modified (usually)
5. **State is crucial** - Terraform tracks infrastructure state to manage resources

---

## Quick Review Questions

1. **Q:** What is the main difference between declarative and imperative IaC?
   **A:** Declarative focuses on the desired end state; imperative focuses on the steps to get there.

2. **Q:** Is Terraform mutable or immutable?
   **A:** Immutable - it replaces resources rather than modifying them in place.

3. **Q:** What is idempotency and why does it matter?
   **A:** Applying the same operation multiple times yields the same result; ensures consistency and safety.

---

[⬅️ Back to Main](../README.md) | [Next: Terraform's Purpose ➡️](02-terraform-purpose.md)
