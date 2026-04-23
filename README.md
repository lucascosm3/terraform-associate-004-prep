# Preparação para Certificação Terraform Associate 004

[![Terraform Version](https://img.shields.io/badge/terraform-%3E%3D%201.0-blue)](https://www.terraform.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> 📚 Repositório pessoal de estudos para o exame **HashiCorp Certified: Terraform Associate (004)**

---

## 🎯 Sobre Este Repositório

Este repositório contém minhas notas de estudo, laboratórios práticos e recursos para preparação da certificação Terraform Associate 004. O objetivo é documentar a jornada de aprendizado e fornecer exemplos práticos para cada objetivo do exame.

**Nota:** Todo o conteúdo técnico (notas, labs e questões) está em inglês, pois a prova será realizada em inglês.

---

## 📋 Visão Geral do Exame

| Detalhe | Informação |
|---------|------------|
| **Certificação** | HashiCorp Certified: Terraform Associate (004) |
| **Duração** | 1 hora |
| **Questões** | 50-60 questões de múltipla escolha, múltiplas respostas, verdadeiro/falso, preenchimento |
| **Nota de Corte** | ~70% |
| **Preço** | $70 USD + impostos |
| **Validade** | 2 anos |
| **Formato** | Online supervisionado (proctored) |

---

## 📖 Objetivos do Exame

O exame cobre **9 domínios principais**:

| # | Domínio | Peso |
|---|---------|------|
| 1 | Conceitos de Infrastructure as Code (IaC) | 11% |
| 2 | Propósito do Terraform (vs outros IaC) | 9% |
| 3 | Fundamentos do Terraform | 24% |
| 4 | CLI do Terraform | 12% |
| 5 | Interação com Módulos Terraform | 15% |
| 6 | Navegação no Workflow do Terraform | 9% |
| 7 | Implementação e Manutenção de State | 12% |
| 8 | Ler, Gerar e Modificar Configurações | 6% |
| 9 | Capacidades do Terraform Cloud e Enterprise | 2% |

---

## 🗂️ Estrutura do Repositório

```
terraform-associate-004-prep/
├── notes/                          # Notas de estudo por tópico (em inglês)
│   ├── 01-infrastructure-as-code.md
│   ├── 02-terraform-purpose.md
│   ├── 03-terraform-basics.md
│   ├── 04-cli-commands.md
│   ├── 05-modules.md
│   ├── 06-workflow.md
│   ├── 07-state-management.md
│   ├── 08-configurations.md
│   └── 09-cloud-enterprise.md
├── labs/                           # Laboratórios práticos do projeto
│   ├── lab-01-first-configuration/
│   ├── lab-02-variables-locals/
│   ├── lab-03-state-remote/
│   ├── lab-04-modules-basics/
│   └── lab-05-workspaces/
├── course-labs/                    # Laboratórios de cursos externos
│   ├── README.md
│   ├── lab-01/
│   ├── lab-02/
│   └── lab-03/
├── exam-topics/                    # Mapeamento dos objetivos do exame
│   └── objectives-checklist.md
├── resources/                      # Recursos externos e links
│   └── useful-links.md
├── README.md                       # Este arquivo
└── LICENSE                         # Licença MIT
```

---

## 🚀 Início Rápido

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
   - Adicione laboratórios de cursos em `course-labs/`
   - Acompanhe o progresso em `exam-topics/objectives-checklist.md`

3. **Execute os laboratórios localmente:**
   ```bash
   cd labs/lab-01-first-configuration
   terraform init
   terraform plan
   terraform apply
   ```

---

## 📝 Cheat Sheet de Comandos Essenciais

```bash
# Inicialização
terraform init                    # Inicializa o diretório de trabalho
terraform init -upgrade          # Atualiza versões dos providers

# Planejamento
terraform plan                    # Visualiza mudanças
terraform plan -out=tfplan        # Salva plano em arquivo
terraform plan -destroy           # Visualiza destruição

# Aplicação
terraform apply                   # Aplica mudanças
terraform apply -auto-approve     # Aplica sem confirmação
terraform apply tfplan            # Aplica plano salvo

# Destruição
terraform destroy                 # Destrói infraestrutura
terraform destroy -auto-approve   # Destrói sem confirmação

# Gerenciamento de State
terraform state list              # Lista recursos no state
terraform state show <resource>   # Mostra detalhes do recurso
terraform state pull              # Visualiza arquivo state bruto

# Workspace
terraform workspace list          # Lista workspaces
terraform workspace new <name>    # Cria novo workspace
terraform workspace select <name> # Muda workspace
terraform workspace delete <name> # Deleta workspace

# Formatação e Validação
terraform fmt                     # Formata arquivos de configuração
terraform validate                # Valida configuração
terraform providers               # Mostra requisitos de providers

# Output e Console
terraform output                  # Mostra outputs
terraform console                 # Console interativo
```

---

## 📚 Recursos Recomendados

### Documentação Oficial
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Language Documentation](https://developer.hashicorp.com/terraform/language)
- [Terraform CLI Documentation](https://developer.hashicorp.com/terraform/cli)

### Prática e Estudo
- [HashiCorp Learn](https://developer.hashicorp.com/terraform/tutorials)
- [Terraform Associate 004 Exam Review](https://developer.hashicorp.com/terraform/tutorials/certification-003)
- [Exam Study Guide](https://developer.hashicorp.com/terraform/tutorials/certification-003/associate-study-003)

### Simulados de Prova (Practice Exams)

O simulado será realizado na Udemy:

📘 **[HashiCorp Certified Terraform Associate 004 - Practice Exams](https://www.udemy.com/home/my-courses/learning/#:~:text=HashiCorp%20Certified%20Terraform%20Associate%20004%20%2D%20Practice%20Exams)**  
por **Bryan Krausen**

- Mesmo autor do curso de preparação
- Múltiplos simulados completos
- Explicações detalhadas das respostas
- Formato semelhante à prova real

---

## ✅ Acompanhamento de Progresso

Acompanhe seu progresso de estudos em [`exam-topics/objectives-checklist.md`](exam-topics/objectives-checklist.md)

---

## 🤝 Contribuição

Este é um repositório pessoal de estudos, mas sugestões e correções são bem-vindas! Sinta-se à vontade para abrir uma issue ou enviar um PR.

---

## 📄 Licença

Este projeto está licenciado sob a Licença MIT - veja o arquivo [LICENSE](LICENSE) para detalhes.

---

## 🎓 Caminho para a Certificação

```
Notas de Estudo → Laboratórios Práticos → Simulados → Passar na Certificação! 🎉
```

**Boa sorte na sua jornada para a certificação Terraform Associate 004! 🚀**
