---
inclusion: fileMatch
fileMatchPattern: ["**/*.tf", "**/*.tfvars", "**/*.tfvars.example", "iac/**"]
---

# Padrões de Terraform (ao gerar ou alterar a IaC)

> Este repositório é um **template** e **não traz a IaC pronta**. Quando o deploy exigir infraestrutura,
> **você gera** os arquivos Terraform em `iac/terraform/` seguindo estes padrões e a referência
> `.kiro/skills/deploy-rest-api/references/03-adaptar-iac.md`.

## Estrutura alvo a criar

```
iac/terraform/
├── backend.tf        # backend S3 (sem lock DynamoDB) + required_providers (aws >= 6.0)
├── providers.tf      # um provider por região, via alias
├── main.tf           # orquestra região(ões) + Route53 (opcional) + dashboard (opcional)
├── variables.tf      # toda variável com type e description
├── outputs.tf        # DNS do ALB, URL do ECR, db_host, nomes de ASG, dashboard
├── backend.hcl.example
├── terraform.tfvars.example
└── modules/
    ├── region-stack/ # VPC, subnets multi-AZ, SGs, ALB+TG+listener, WAF, ECR, RDS, S3, IAM, ASG
    └── dashboard/    # CloudWatch (opcional)
```

Use **um módulo reutilizável por região**. Diferenças entre regiões entram como flag
(`create_database`, `create_s3_bucket`), não como cópia de recursos.

## Regras

1. **Toda variável** tem `type` e `description`. Default seguro; nenhum segredo em default.
2. **Nomes** prefixados por `var.environment_name`. Em recursos que podem ser recriados
   (Security Group, Launch Template, Target Group, Role, Instance Profile) use `name_prefix` +
   `lifecycle { create_before_destroy = true }`.
3. **Tags**: todo recurso recebe um `local.common_tags`.
4. **Segredos**: senha de banco com `random_password` + `aws_secretsmanager_secret_version`.
   A instância lê o segredo em runtime (UserData); nunca receba senha no código.
5. **Provider AWS `>= 6.0`**: use `data.aws_region.current.region` (o atributo `.name` está deprecado).
6. **State**: backend S3 por `-backend-config` (bucket vem de `TF_STATE_BUCKET`). Sem lock DynamoDB:
   evite dois `apply` simultâneos. Nunca versione `backend.hcl`, `*.tfstate` ou `*.tfvars`.
7. **Formato**: rode `terraform fmt -recursive` e, com `terraform init -backend=false`, `terraform validate`
   a cada alteração. Não commite com erro de fmt/validate.
8. **Itens didáticos** (SG `0.0.0.0/0`, RDS público, SSH aberto, `force_destroy`, `skip_final_snapshot`)
   precisam de um comentário `# Didatico:` com o motivo e o risco.
9. **Recursos opcionais** (RDS, S3, Route53, WAF, dashboard, segunda região) ligam/desligam por variável.
   Prefira adicionar uma flag a remover código.

## Limites

- Você valida e formata localmente, mas **não aplica**. O `plan`/`apply` rodam no GitHub Actions.
- Alterações que **destroem ou substituem** recursos de dados (RDS, S3, Secret) exigem aviso explícito
  ao dev e menção no corpo do PR.
