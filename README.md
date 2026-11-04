# Cloud_and_AI

Template para **publicar uma API REST (com Dockerfile) na AWS** com **Terraform** e **GitHub Actions**,
com o **Kiro guiando cada passo** e **guardrails** que limitam o que o agente pode fazer.

Este repositório **não contém código de aplicação**. Ele traz só o necessário para o Kiro trabalhar
em infraestrutura, deploy e pipeline.

## Como usar

1. Use este repositório como template (ou copie `.kiro/`, `.github/`, `iac/` e `docs/`) e adicione a sua API
   na raiz, com um `Dockerfile`.
2. Faça a preparação única (bucket de state, usuário IAM, secrets): [`docs/kiro-deploy/01-preparacao.md`](docs/kiro-deploy/01-preparacao.md).
3. No Kiro, com o repositório aberto, digite:

   ```
   /deploy-rest-api
   ```

   O Kiro mostra o mapa de fases, faz a entrevista, valida a API, adapta a IaC, abre o PR e acompanha o
   pipeline. Ele para em pontos de aprovação e **nunca aplica infraestrutura nem vê credenciais AWS**.

## O que tem aqui

```
.kiro/
├── steering/        # regras e contexto (contexto, guardrails, terraform, actions, contrato da API, IAM)
├── skills/          # /deploy-rest-api: roteiro de 8 fases com gates, referências e templates
├── agents/          # deploy-guide: agente com permissões allow/ask/deny
├── hooks/           # bloqueios automáticos (IDE e CLI)
└── settings/        # MCP: documentação oficial da AWS (somente leitura)

.github/
├── workflows/       # terraform-ci, terraform-plan, deploy/destroy da infra, deploy do backend, schema do banco
├── pull_request_template.md
└── CODEOWNERS

iac/
├── terraform/       # IaC de referência (módulo por região: VPC, ALB, ASG, WAF, ECR, RDS, S3, dashboard)
└── scripts/         # apply, destroy, refresh-instances

docs/kiro-deploy/    # guia para o dev: preparação, fluxo, guardrails, nova API, roteiro de palestra
```

## Antes do primeiro uso, revise

- **Valores de exemplo** em `iac/terraform/variables.tf` (ambiente, ECR, **domínio**, banco). Detalhes em
  [`iac/terraform/README.md`](iac/terraform/README.md).
- **`.github/CODEOWNERS`**: troque pelo seu usuário.
- **Workflow `Deploy Backend API`** roda a cada push na `main`. Sem `Dockerfile` e sem secrets configurados,
  esse run falha. É esperado até a preparação ser concluída.

## Documentação

Comece por [`docs/kiro-deploy/README.md`](docs/kiro-deploy/README.md).
