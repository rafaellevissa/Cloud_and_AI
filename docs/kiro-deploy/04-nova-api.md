# 04. Usando o kit com a sua API

## O que a sua API precisa

Resumo do contrato (completo em `.kiro/steering/40-contrato-da-api.md`):

1. `Dockerfile` na raiz que inicia o servidor, com `EXPOSE <porta>`.
2. `GET /` (ou outro path) respondendo **200**, sem autenticação e **sem consultar o banco**.
3. Configuração 100% por **variáveis de ambiente**; nenhum segredo na imagem.
4. **Stateless** (haverá 2+ réplicas atrás do ALB).
5. Imagem **linux/amd64**.
6. Recomendado: logs JSON em stdout com `type: "request"` e `userAgent` (alimenta o dashboard).

## Como informar ao Kiro

O Kiro cria o `deploy-config.yaml` perguntando. Se preferir, copie
`.kiro/skills/deploy-rest-api/assets/deploy-config.example.yaml` para a raiz e preencha. O arquivo
**não contém segredos**.

## O que o Kiro gera a partir do `deploy-config.yaml`

Rede em 2 AZs, ALB, Auto Scaling Group, WAF, ECR, RDS MySQL (opcional), Secrets Manager, S3 (opcional),
logs e dashboard no CloudWatch, DNS (opcional), e os pipelines de infra, build e redeploy. Tudo é **gerado
durante o fluxo** (fases 3 e 4), não vem pronto no template.

## O que varia conforme a sua API

| Item | De onde o Kiro tira |
|---|---|
| Nome do ECR | `app.name` / `infra` |
| Porta do container | `app.container_port` |
| Path do health check | `app.health_check_path` |
| Variáveis de ambiente | `app.env_vars` |
| Banco / S3 / DNS / regiões | flags em `infra.*` |

Como isso vira Terraform: `.kiro/skills/deploy-rest-api/references/03-adaptar-iac.md`.

## O que ainda não é automático (transparência)

- O banco de referência é **MySQL/RDS**. Outro banco exige o Kiro ajustar o módulo (sinalize isso; pode
  virar uma spec).
- O **schema do banco** é aplicado por um workflow separado que assume **API Node.js** (`npm run db:schema`).
  Para outras stacks, o Kiro adapta o workflow na fase 3.
- A arquitetura de referência é multi-região. **Uma região só** é um caso válido e mais barato: o Kiro gera
  apenas o módulo primário, sem registro DNS secundário nem widgets extras.

## Mais de uma API no mesmo repositório?

O fluxo assume **uma API por repositório** (um state por API). Para várias APIs, use um repositório por API.
