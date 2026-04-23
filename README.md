# Preparação para Certificação Terraform Associate 004

[![Terraform Version](https://img.shields.io/badge/terraform-%3E%3D%201.0-blue)](https://www.terraform.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> Repositório pessoal de estudos para o exame **HashiCorp Certified: Terraform Associate (004)**

---

## Sobre Este Repositório

Este repositório contém minhas notas de estudo, laboratórios práticos e recursos para preparação da certificação Terraform Associate 004. O objetivo é documentar a jornada de aprendizado e fornecer exemplos práticos para cada objetivo do exame.

**Nota:** Todo o conteúdo técnico (notas, labs e questões) está em inglês, pois a prova será realizada em inglês.

---

## Visão Geral do Exame

| Detalhe | Informação |
|---------|------------|
| **Certificação** | HashiCorp Certified: Terraform Associate (004) |
| **Duração** | 1 hora |
| **Questões** | 50-60 questões de múltipla escolha, múltiplas respostas, verdadeiro/falso, preenchimento |
| **Nota de Corte** | ~70% |
| **Preço** | $70 USD + impostos |
| **Validade** | 2 anos |
| **Formato** | Online supervisionado (proctored) |
| **Versão do Terraform** | 1.12 |

---

## Domínios do Exame (004)

O exame 004 cobre **8 domínios** (diferente da versão 003 que tinha 9):

| # | Domínio | Peso |
|---|---------|------|
| 1 | Infrastructure as Code (IaC) with Terraform | ~11% |
| 2 | Terraform Fundamentals (Providers + State) | ~12% |
| 3 | Core Terraform Workflow (Init/Plan/Apply/Destroy/Fmt/Validate) | ~9% |
| 4 | Terraform Configuration | ~22% |
| 5 | Terraform Modules | ~11% |
| 6 | Terraform State Management | ~13% |
| 7 | Maintain Infrastructure (Import, State CLI, Logging) | ~5% |
| 8 | HCP Terraform | ~5% |

### Mudanças entre 003 e 004

- Domínios foram reorganizados de 9 para 8
- "Terraform Fundamentals" agora inclui state (além de providers)
- "Core Workflow" agora inclui `terraform fmt` e `terraform validate`
- "Terraform Configuration" agora inclui: custom conditions (preconditions/postconditions), checks, sensitive data, Vault, `moved` block, `removed` block
- "State Management" agora inclui: resource drift, refresh-only mode, `moved` block, `removed` block
- Novo domínio: "Maintain Infrastructure" (import, state CLI inspection, verbose logging)
- "Terraform Cloud" renomeado para "HCP Terraform" (inclui Explorer, Change requests, Private Registry, Dynamic provider credentials, etc.)
- `terraform taint` está **depreciado** — usar `terraform apply -replace=`
- `terraform refresh` foi **substituído** por `terraform apply -refresh-only`

---

## Estrutura do Repositório

```
terraform-associate-004-prep/
├── notes/                              # Notas de estudo por domínio (em inglês)
│   ├── 01-iac-concepts.md              # Domain 1: IaC with Terraform
│   ├── 02-terraform-fundamentals.md    # Domain 2: Providers + State
│   ├── 03-core-workflow.md             # Domain 3: Init/Plan/Apply/Destroy/Fmt/Validate
│   ├── 04-terraform-configuration.md   # Domain 4: Configuration (BIGGEST domain)
│   ├── 05-modules.md                   # Domain 5: Modules
│   ├── 06-state-management.md          # Domain 6: State Management
│   ├── 07-maintain-infrastructure.md   # Domain 7: Import, State CLI, Logging
│   └── 08-hcp-terraform.md            # Domain 8: HCP Terraform
├── labs/                               # Laboratórios práticos do projeto
│   ├── lab-01-first-configuration/
│   ├── lab-02-variables-locals/
│   ├── lab-03-state-remote/
│   ├── lab-04-modules-basics/
│   └── lab-05-workspaces/
├── course-labs/                         # Laboratórios de cursos externos
│   ├── README.md
│   ├── lab-01/
│   ├── lab-02/
│   └── lab-03/
├── exam-topics/                         # Mapeamento dos objetivos do exame
│   └── objectives-checklist.md
├── resources/                           # Recursos externos e links
│   └── useful-links.md
├── README.md                            # Este arquivo
└── LICENSE                              # Licença MIT
```

---

## Início Rápido

### Pré-requisitos

- [Terraform CLI](https://developer.hashicorp.com/terraform/install) (v1.0+)
- Editor de texto ou IDE (recomendado VS Code com extensão Terraform)
- Conta de provedor cloud (AWS, Azure ou GCP) para os laboratórios

### Usando Este Repositório

1. **Clone o repositório:**
   ```bash
   git clone https://github.com/lucascosm3/terraform-associate-004-prep.git
   cd terraform-associate-004-prep
   ```

2. **Navegue pelos tópicos:**
   - Comece com `notes/` para conceitos teóricos
   - Pratique com laboratórios em `labs/`
   - Acompanhe o progresso em `exam-topics/objectives-checklist.md`

3. **Execute os laboratórios localmente:**
   ```bash
   cd labs/lab-01-first-configuration
   terraform init
   terraform plan
   terraform apply
   ```

---

## Cheat Sheet de Comandos Essenciais

```bash
# Inicialização
terraform init                    # Inicializa o diretório de trabalho
terraform init -upgrade          # Atualiza versões dos providers

# Validação e Formatação
terraform fmt                     # Formata arquivos de configuração
terraform fmt -check              # Verifica se arquivos estão formatados (CI)
terraform validate                # Valida configuração

# Planejamento
terraform plan                    # Visualiza mudanças
terraform plan -out=tfplan        # Salva plano em arquivo
terraform plan -destroy           # Visualiza destruição
terraform plan -detailed-exitcode # Exit codes: 0=sem mudanças, 1=erro, 2=mudanças

# Aplicação
terraform apply                   # Aplica mudanças
terraform apply -auto-approve     # Aplica sem confirmação
terraform apply tfplan            # Aplica plano salvo
terraform apply -replace=aws_instance.web  # Substitui recurso (substitui taint)

# Destruição
terraform destroy                 # Destrói infraestrutura

# Refresh-only (substitui terraform refresh)
terraform apply -refresh-only     # Atualiza state sem fazer mudanças

# Gerenciamento de State
terraform state list              # Lista recursos no state
terraform state show <resource>   # Mostra detalhes do recurso
terraform state pull              # Visualiza arquivo state bruto

# Import
terraform import <type>.<name> <id>  # Importa recurso existente

# Workspace
terraform workspace list          # Lista workspaces
terraform workspace new <name>    # Cria novo workspace
terraform workspace select <name> # Muda workspace

# Output e Console
terraform output                  # Mostra outputs
terraform console                 # Console interativo

# Logging
TF_LOG=DEBUG terraform apply     # Logging verboso
TF_LOG_PATH=/tmp/tf.log terraform apply  # Log para arquivo
```

---

## Recursos Recomendados

### Documentação Oficial
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Language Documentation](https://developer.hashicorp.com/terraform/language)
- [Terraform CLI Documentation](https://developer.hashicorp.com/terraform/cli)

### Prática e Estudo
- [HashiCorp Learn](https://developer.hashicorp.com/terraform/tutorials)
- [Terraform Associate 004 Exam Review](https://developer.hashicorp.com/terraform/tutorials/certification-004)

### Simulados de Prova (Practice Exams)

📘 **[HashiCorp Certified Terraform Associate 004 - Practice Exams](https://www.udemy.com/home/my-courses/learning/#:~:text=HashiCorp%20Certified%20Terraform%20Associate%20004%20%2D%20Practice%20Exams)**
por **Bryan Krausen**

- Mesmo autor do curso de preparação
- Múltiplos simulados completos
- Explicações detalhadas das respostas
- Formato semelhante à prova real

---

## Acompanhamento de Progresso

Acompanhe seu progresso de estudos em [`exam-topics/objectives-checklist.md`](exam-topics/objectives-checklist.md)

---

## Contribuição

Este é um repositório pessoal de estudos, mas sugestões e correções são bem-vindas! Sinta-se à vontade para abrir uma issue ou enviar um PR.

---

## Licença

Este projeto está licenciado sob a Licença MIT - veja o arquivo [LICENSE](LICENSE) para detalhes.

---

## Caminho para a Certificação

```
Notas de Estudo → Laboratórios Práticos → Simulados → Passar na Certificação!
```

**Boa sorte na sua jornada para a certificação Terraform Associate 004!**