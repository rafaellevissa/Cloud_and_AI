# 03. Gerar a IaC para a API

Este template **não traz a IaC pronta**. Nesta fase você **gera** os arquivos Terraform em
`iac/terraform/` a partir do `deploy-config.yaml` e dos padrões de `.kiro/steering/20-terraform.md`.
Seja transparente com o dev: explique que está criando a infraestrutura do zero, com o menor conjunto
de recursos que atende ao que foi pedido.

## Arquitetura de referência a gerar

`Route53 (opcional) → ALB (+WAF) → Auto Scaling Group (EC2 em 2+ AZs, roda o container) → RDS MySQL`,
mais ECR, S3 (opcional), Secrets Manager, CloudWatch (logs + dashboard). Estrutura de arquivos e regras:
`.kiro/steering/20-terraform.md`.

## O que é padrão (gere sempre, igual)

VPC, subnets em 2 AZs, Internet Gateway, Security Groups, ALB + Target Group + Listener,
Launch Template + ASG com instance refresh, log group, role/instance profile, ECR.

## O que você define a partir do `deploy-config.yaml`

| O quê | Origem no `deploy-config.yaml` | Como gerar |
|---|---|---|
| Nome do ECR | `infra` / `app.name` | Variável `ecr_repository_name`; **igual** ao secret `ECR_REPOSITORY` |
| Porta do container | `app.container_port` | `docker run -p 80:<porta>` no `user-data.sh.tftpl` (via `templatefile`) |
| Health check | `app.health_check_path` | `health_check { path = ... }` no target group |
| Variáveis de ambiente | `app.env_vars` | Bloco `docker run -e ...` no `user-data.sh.tftpl` |
| Banco (MySQL) | `infra.database.enabled` | Se `false`: não crie RDS/Secret e não injete `DB_*` na UserData |
| S3 | `infra.object_storage.enabled` | Crie o bucket só se `true` |
| DNS | `infra.dns.enabled` | `aws_route53_record` condicional; a hosted zone já deve existir |
| Regiões | `infra.primary_region` / `secondary_regions` | Uma região só: gere apenas o módulo primário, sem registro Route53 secundário nem widgets extras |
| Tamanho/quantidade | `infra.instance_type` / `infra.asg.*` | Variáveis |
| WAF | `infra.waf` | Common Rule Set; só adicione a regra de corpo grande se houver endpoint de upload |

## Procedimento

1. **Mapeie** `deploy-config.yaml` → variáveis/flags. Liste o que vai gerar.
2. **Escolha o caminho:**
   - *Direto* quando a arquitetura de referência atende sem grandes desvios.
   - *Spec* (`.kiro/specs/<feature>/` com requirements → design → tasks) quando houver decisões de
     arquitetura relevantes (trocar banco, múltiplas regiões, outro tipo de compute).
3. **Parametrize em vez de fixar valores.** Porta, health check e env vars vêm de variáveis, não
   hardcoded, para a próxima API só precisar trocar o `deploy-config.yaml`.
4. Gere também `terraform.tfvars.example`, `backend.hcl.example` e um `README.md` curto da IaC.
5. `terraform fmt -recursive` + `terraform init -backend=false` + `terraform validate`.
6. Apresente o resumo (**G3**): arquivos gerados, recursos criados, itens didáticos presentes e o que
   gera custo contínuo.

## Mudanças que destroem dados

Ao **alterar** uma IaC já aplicada, qualquer mudança que force **substituição** do RDS, do Secret ou do
bucket (renomear, trocar engine, mudar identificador) apaga dados. Sinalize em destaque no PR e confirme.

## Atenção: mudar `user-data` recria instâncias

A UserData faz parte do Launch Template. Alterá-la e aplicar dispara *instance refresh* (rolling) no ASG.
É esperado, mas informe o dev que haverá troca de instâncias.
