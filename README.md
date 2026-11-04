# Cloud_and_AI

Instruções para o **Kiro** publicar uma **API REST (com Dockerfile) na AWS** com **Terraform** e
**GitHub Actions**, com **guardrails** que limitam o que o agente pode fazer.

Este repositório contém **apenas** o necessário para o Kiro atuar: o material em `.kiro/` e a documentação
em `docs/`. **Não há código de aplicação, nem IaC, nem workflows prontos** — a infraestrutura (`iac/`) e
os pipelines (`.github/`) são **gerados pelo Kiro** durante o fluxo de deploy.

## Como usar

1. Use este repositório como template (ou copie `.kiro/` e `docs/`) e adicione a sua API na raiz,
   com um `Dockerfile`.
2. Faça a preparação única (bucket de state, usuário IAM, secrets): [`docs/kiro-deploy/01-preparacao.md`](docs/kiro-deploy/01-preparacao.md).
3. No Kiro, com o repositório aberto, digite:

   ```
   /deploy-rest-api
   ```

   O Kiro mostra o mapa de fases, faz a entrevista, valida a API, **gera a IaC e os workflows**, abre o PR
   e acompanha o pipeline. Ele para em pontos de aprovação e **nunca aplica infraestrutura nem vê
   credenciais AWS**.

## O que tem aqui

```
.kiro/
├── steering/        # regras e contexto (contexto, guardrails, terraform, actions, contrato da API, IAM)
├── skills/          # /deploy-rest-api: roteiro de 8 fases com gates, referências e templates
├── agents/          # deploy-guide: agente com permissões allow/ask/deny
├── hooks/           # bloqueios automáticos (IDE e CLI)
└── settings/        # MCP: documentação oficial da AWS (somente leitura)

docs/kiro-deploy/    # guia para o dev: preparação, fluxo, guardrails, nova API, roteiro de palestra
```

### Gerado pelo Kiro durante o fluxo (não existe no template)

```
iac/terraform/       # IaC (módulo por região: VPC, ALB, ASG, WAF, ECR, RDS, S3, dashboard)
.github/workflows/   # terraform-ci, terraform-plan, deploy/destroy da infra, deploy do backend, schema
.github/pull_request_template.md, .github/CODEOWNERS
```

Esses arquivos aparecem no Pull Request que o Kiro abre; eles não ficam versionados neste template.

## Documentação

Comece por [`docs/kiro-deploy/README.md`](docs/kiro-deploy/README.md).
