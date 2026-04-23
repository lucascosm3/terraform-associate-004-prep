# Labs do Curso

Esta pasta contém os laboratórios práticos do curso que você está realizando.

---

## 📚 Estrutura

Cada laboratório do curso deve ser colocado em sua própria pasta:

```
course-labs/
├── README.md                 # Este arquivo
├── lab-01/                   # Laboratório 1 do curso
│   ├── README.md            # Notas e comandos do lab
│   ├── main.tf              # Código Terraform
│   ├── variables.tf         # Variáveis
│   └── outputs.tf           # Outputs
├── lab-02/                   # Laboratório 2 do curso
│   └── ...
└── lab-03/
    └── ...
```

---

## 📝 Template para Novos Labs

Crie uma nova pasta para cada lab do curso:

```bash
mkdir course-labs/lab-XX
cd course-labs/lab-XX
```

### Arquivos recomendados:

**README.md** - Anotações do lab:
```markdown
# Lab XX: [Título do Lab]

## 🎯 Objetivo
[Descreva o objetivo do lab]

## 📝 Comandos Importantes
```bash
terraform init
terraform plan
terraform apply
```

## 💡 Aprendizados Principais
- [Pontos importantes aprendidos]
- [Comandos novos]
- [Conceitos-chave]

## ❓ Dúvidas
- [Anote suas dúvidas aqui para revisar depois]

## 🔗 Referências do Curso
- Módulo: [nome do módulo]
- Timestamp: [minuto do vídeo]
```

---

## 🗂️ Lista de Labs do Curso

Atualize esta lista conforme avança no curso:

| # | Título | Status | Data |
|---|--------|--------|------|
| 01 | [Adicionar título] | ⬜ Não iniciado | - |
| 02 | [Adicionar título] | ⬜ Não iniciado | - |
| 03 | [Adicionar título] | ⬜ Não iniciado | - |

**Legenda:**
- ⬜ Não iniciado
- 🟡 Em andamento
- ✅ Concluído

---

## 💡 Dicas

1. **Faça commits frequentes** - Salve seu progresso no Git
2. **Documente suas dúvidas** - Anote no README de cada lab
3. **Compare com os labs do projeto** - Veja `labs/` para referência adicional
4. **Experimente variações** - Não tenha medo de modificar o código

---

## 🚀 Comandos Úteis

```bash
# Inicializar novo lab
cd course-labs/lab-XX
terraform init

# Verificar formatação
terraform fmt

# Validar configuração
terraform validate

# Salvar no Git
git add course-labs/lab-XX/
git commit -m "course: adiciona lab XX - [tema]"
```

---

**Bons estudos! 🎓**
