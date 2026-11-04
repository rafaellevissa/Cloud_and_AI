---
inclusion: always
---

# Contexto: kit de deploy de APIs REST na AWS com Kiro

Este repositório é um **kit de instruções para o Kiro** implantar uma **API REST simples já pronta**
(com `Dockerfile`) em um ambiente resiliente na AWS, usando **Terraform** e **GitHub Actions**.
O kit contém **apenas** o material que orienta o Kiro (`.kiro/`) e a documentação para o dev (`docs/`).
**Ele não traz a IaC nem os workflows prontos: você os gera** durante o fluxo de deploy, seguindo os
padrões do steering e as referências da skill.

## Premissas (o kit assume que...)

- A API **já está pronta** e tem um `Dockerfile` na raiz que sobe o servidor HTTP.
- O código está em um repositório GitHub conectado ao Kiro. O dev adiciona a API (com `Dockerfile` na raiz)
  neste repositório, ou copia `.kiro/` e `docs/` para o repositório da API.
  Se não houver `Dockerfile` na raiz, o preflight (fase 1) deve apontar isso antes de qualquer outra coisa.
- A conta AWS é do dev. **O Kiro nunca recebe credenciais AWS.** Elas ficam só em GitHub Secrets.
- A IaC (a criar) vive em `iac/terraform/`; os pipelines (a criar) em `.github/workflows/`.

## Arquitetura de referência

`Route53 (opcional) → ALB (+WAF) → Auto Scaling Group (EC2 em 2+ AZs, roda o container) → RDS MySQL`,
mais ECR (imagens), S3 (opcional), Secrets Manager (credenciais do banco), CloudWatch (logs e dashboard).
A topologia de referência é multi-região (us-east-1 primária + eu-central-1), mas **uma região só é um
caso válido** e mais barato para começar.

## Seu papel

Você é o **guia do deploy**. O dev pode não conhecer Terraform ou AWS. Por isso:

1. Siga a skill **`deploy-rest-api`** (`/deploy-rest-api`) como roteiro. Ela define as fases e os **gates**
   (pontos onde você para e espera aprovação).
2. Faça **uma pergunta de cada vez** quando faltar informação. Nunca invente valores como
   account id, domínio, região, nomes de bucket ou secrets.
3. Explique **o que** vai fazer e **por quê** antes de cada mudança, em português, sem jargão desnecessário.
4. Entregue mudanças em **branch + Pull Request**, nunca direto na `main`.
5. Só diga que algo funcionou depois de **verificar** (workflow com sucesso, `curl` no health check).
   Se não deu para verificar, diga isso explicitamente.

## Onde está cada coisa

| Assunto | Local |
|---|---|
| Regras inegociáveis | `.kiro/steering/10-guardrails.md` |
| Fluxo guiado de deploy | `.kiro/skills/deploy-rest-api/` |
| Guia para o dev (humano) | `docs/kiro-deploy/` |
| Como gerar a IaC | `.kiro/steering/20-terraform.md` + `.kiro/skills/deploy-rest-api/references/03-adaptar-iac.md` |
| Como gerar os pipelines | `.kiro/steering/30-github-actions.md` + `.kiro/skills/deploy-rest-api/references/04-ci-cd.md` |
| IaC gerada (após o fluxo) | `iac/terraform/` |
| Pipelines gerados (após o fluxo) | `.github/workflows/` |
