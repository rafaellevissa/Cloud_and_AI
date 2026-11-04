---
inclusion: fileMatch
fileMatchPattern: [".github/workflows/**", "iac/scripts/**"]
---

# Padrões dos pipelines (ao gerar ou alterar workflows)

> Este template **não traz os workflows prontos**. Quando o deploy exigir CI/CD, **você gera** os
> arquivos em `.github/workflows/` seguindo estes padrões. Referência:
> `.kiro/skills/deploy-rest-api/references/04-ci-cd.md`.

## Workflows alvo a criar

| Workflow | Disparo | O que faz |
|---|---|---|
| `terraform-ci.yml` | Pull Request | `terraform fmt -check` + `validate` + guarda contra segredos versionados. **Sem credenciais AWS.** |
| `terraform-plan.yml` | manual | `terraform plan` com resumo (criar/alterar/destruir). **Não aplica.** |
| `deploy-infraestructure.yml` | manual | `terraform apply` **em paralelo** com build/push da imagem |
| `deploy-backend.yml` | push na `main` + `workflow_call` | build → push no ECR → rolling instance refresh |
| `destroy-infrastructure.yml` | manual | `terraform destroy` (somente humano dispara) |
| `apply-db-schema.yml` | manual | abre acesso temporário ao RDS, aplica o schema, fecha o acesso |

## Secrets esperados

| Secret | Uso |
|---|---|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Usuário IAM do pipeline |
| `TF_STATE_BUCKET` | Bucket S3 do state do Terraform |
| `ECR_REGISTRY` / `ECR_REPOSITORY` | Destino do `docker push` |

Você **não cria** secrets. Liste o que falta e peça ao dev para cadastrar em
*Settings → Secrets and variables → Actions*.

## Regras

1. **Workflows de infraestrutura são `workflow_dispatch`** (disparo manual). Não use `push` para eles.
2. **Ordem do deploy:** `deploy-infrastructure` e `build-and-push-docker-image` rodam **em paralelo**;
   o build espera ~20s (`wait-for-ecr: true`) para o Terraform criar o ECR. **Não** transforme isso em
   `needs:` sem perguntar ao dev. Se o build falhar por ECR inexistente, sugira aumentar o delay ou
   reexecutar o job, e explique a corrida.
3. `refresh-instances` depende do build (`needs: build-and-image-push`).
4. **Pin de versões**: `hashicorp/setup-terraform` com `terraform_version` fixa.
5. **Nunca** imprima segredos. Use `::add-mask::` em valores sensíveis lidos em runtime.
6. Passos que abrem acesso temporário (ex.: liberar o IP do runner no SG do RDS) **sempre** fecham o
   acesso em um passo com `if: always()`.
7. Valide o YAML antes do commit e confira que `needs`/`if` fazem o que você descreveu no PR.
8. Referencie workflows reutilizáveis **localmente** (`./.github/workflows/...`), nunca apontando para
   outro repositório.

## Quando algo falhar

Consulte `.kiro/skills/deploy-rest-api/references/05-troubleshooting.md`. Para ler logs:
`gh run list` e `gh run view <id> --log-failed`. Resuma a causa em português e proponha a correção.
