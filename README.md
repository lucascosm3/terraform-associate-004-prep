# Terraform Associate 004 Certification Prep

[![Terraform Version](https://img.shields.io/badge/terraform-%3E%3D%201.0-blue)](https://www.terraform.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> 📚 Personal study repository for the **HashiCorp Certified: Terraform Associate (004)** exam

---

## 🎯 About This Repository

This repository contains my notes, hands-on labs, and study resources for preparing the Terraform Associate 004 certification. The goal is to document the learning journey and provide practical examples for each exam objective.

---

## 📋 Exam Overview

| Detail | Information |
|--------|-------------|
| **Certification** | HashiCorp Certified: Terraform Associate (004) |
| **Duration** | 1 hour |
| **Questions** | 50-60 multiple choice, multiple answer, true/false, fill in the blank |
| **Passing Score** | ~70% |
| **Price** | $70 USD + tax |
| **Validity** | 2 years |
| **Format** | Online proctored |

---

## 📖 Exam Objectives

The exam covers **9 major domains**:

| # | Domain | Weight |
|---|--------|--------|
| 1 | Infrastructure as Code (IaC) Concepts | 11% |
| 2 | Terraform's Purpose (vs other IaC) | 9% |
| 3 | Terraform Basics | 24% |
| 4 | Terraform CLI | 12% |
| 5 | Interact with Terraform Modules | 15% |
| 6 | Navigate Terraform Workflow | 9% |
| 7 | Implement and Maintain State | 12% |
| 8 | Read, Generate, Modify Configurations | 6% |
| 9 | Understand Terraform Cloud and Enterprise Capabilities | 2% |

---

## 🗂️ Repository Structure

```
terraform-associate-004-prep/
├── notes/                          # Study notes by topic
│   ├── 01-infrastructure-as-code.md
│   ├── 02-terraform-purpose.md
│   ├── 03-terraform-basics.md
│   ├── 04-cli-commands.md
│   ├── 05-modules.md
│   ├── 06-workflow.md
│   ├── 07-state-management.md
│   ├── 08-configurations.md
│   └── 09-cloud-enterprise.md
├── labs/                           # Hands-on practice labs (project)
│   ├── lab-01-first-configuration/
│   ├── lab-02-variables-locals/
│   ├── lab-03-state-remote/
│   ├── lab-04-modules-basics/
│   └── lab-05-workspaces/
├── course-labs/                    # Labs from external courses
│   ├── README.md
│   ├── lab-01/
│   ├── lab-02/
│   └── lab-03/
├── exam-topics/                    # Exam objectives mapping
│   └── objectives-checklist.md
├── resources/                      # External resources & links
│   └── useful-links.md
├── README.md                       # This file
└── LICENSE                         # MIT License
```

---

## 🚀 Quick Start

### Prerequisites

- [Terraform CLI](https://developer.hashicorp.com/terraform/install) (v1.0+)
- Text editor or IDE (VS Code with Terraform extension recommended)
- Cloud provider account (AWS, Azure, or GCP) for labs

### Using This Repository

1. **Clone the repository:**
   ```bash
   git clone https://github.com/lucascosm3/terraform-associate-004-prep.git
   cd terraform-associate-004-prep
   ```

2. **Navigate through topics:**
   - Start with `notes/` for theoretical concepts
   - Practice with hands-on labs in `labs/`
   - Add labs from external courses in `course-labs/`
   - Track progress using `exam-topics/objectives-checklist.md`

3. **Run labs locally:**
   ```bash
   cd labs/lab-01-first-configuration
   terraform init
   terraform plan
   terraform apply
   ```

---

## 📝 Essential Commands Cheat Sheet

```bash
# Initialization
terraform init                    # Initialize working directory
terraform init -upgrade          # Upgrade provider versions

# Planning
terraform plan                    # Preview changes
terraform plan -out=tfplan        # Save plan to file
terraform plan -destroy           # Preview destruction

# Applying
terraform apply                   # Apply changes
terraform apply -auto-approve     # Apply without confirmation
terraform apply tfplan            # Apply saved plan

# Destroying
terraform destroy                 # Destroy infrastructure
terraform destroy -auto-approve   # Destroy without confirmation

# State Management
terraform state list              # List resources in state
terraform state show <resource>   # Show resource details
terraform state pull              # View raw state file

# Workspace
terraform workspace list          # List workspaces
terraform workspace new <name>    # Create new workspace
terraform workspace select <name> # Switch workspace
terraform workspace delete <name> # Delete workspace

# Formatting & Validation
terraform fmt                     # Format configuration files
terraform validate                # Validate configuration
terraform providers               # Show provider requirements

# Output & Console
terraform output                  # Show outputs
terraform console                 # Interactive console
```

---

## 📚 Recommended Resources

### Official Documentation
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Language Documentation](https://developer.hashicorp.com/terraform/language)
- [Terraform CLI Documentation](https://developer.hashicorp.com/terraform/cli)

### Practice & Study
- [HashiCorp Learn](https://developer.hashicorp.com/terraform/tutorials)
- [Terraform Associate 004 Exam Review](https://developer.hashicorp.com/terraform/tutorials/certification-003)
- [Exam Study Guide](https://developer.hashicorp.com/terraform/tutorials/certification-003/associate-study-003)

### Practice Exams
- Whizlabs Terraform Associate Practice Tests
- Udemy - Terraform Practice Exams (Bryan Krausen)

---

## ✅ Progress Tracker

Track your study progress in [`exam-topics/objectives-checklist.md`](exam-topics/objectives-checklist.md)

---

## 🤝 Contributing

This is a personal study repository, but suggestions and corrections are welcome! Feel free to open an issue or submit a PR.

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🎓 Certification Path

```
Study Notes → Hands-on Labs → Practice Exams → Pass Certification! 🎉
```

**Good luck with your Terraform Associate 004 journey! 🚀**
