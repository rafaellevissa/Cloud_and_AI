# Infraestrutura (Terraform)

IaC de referência para publicar uma **API REST em contêiner** em um ambiente resiliente na AWS.
Topologia multi-região (uma região só também é um caso válido; ver `docs/kiro-deploy/04-nova-api.md`):

- **us-east-1** (região primária): VPC, ALB, ASG (multi-AZ), WAF, ECR, **RDS MySQL**,
  **Secrets Manager**, **S3** (bucket global), CloudWatch Log Group.
- **eu-central-1** (região secundária): mesma stack de compute/rede, consumindo o
  banco, o secret e o bucket S3 da região primária.
- **Route53**: registro `A` com *latency routing* apontando para o ALB de cada região.
- **CloudWatch Dashboard**: requests por ALB, CPU por ASG e requests por instância.

Tudo é provisionado por um **único `terraform apply`** (uma raiz com dois providers
AWS por alias de região).

## Estrutura

```
iac/terraform/
├── backend.tf                 # backend S3 (sem lock DynamoDB) + required_providers
├── providers.tf               # providers aws.us_east_1 e aws.eu_central_1
├── main.tf                    # orquestra as 2 regiões + Route53 + dashboard
├── variables.tf               # variáveis da raiz (com defaults)
├── outputs.tf                 # outputs (DNS, ECR, DB, dashboard, ASGs...)
├── backend.hcl.example        # modelo de config do backend S3
├── terraform.tfvars.example   # modelo de variáveis
└── modules/
    ├── region-stack/          # módulo reutilizável por região
    │   ├── networking.tf      # VPC, subnets multi-AZ, IGW, route table, SGs
    │   ├── data.tf            # RDS + Secret + ECR + S3 (condicionais por flag)
    │   ├── iam.tf             # roles/instance profile + log group
    │   ├── loadbalancer.tf    # ALB + target group + listener
    │   ├── waf.tf             # WAFv2 (Common Rule Set + regra custom /upload)
    │   ├── compute.tf         # launch template + ASG + instance refresh
    │   └── templates/user-data.sh.tftpl
    └── dashboard/             # CloudWatch dashboard multi-região
```

As duas regiões reusam o mesmo módulo `region-stack`; a diferença é só o valor das flags
(`create_database`, `create_s3_bucket`) e o provider passado.

## Valores de exemplo que você deve revisar

Os defaults de `variables.tf` vêm do projeto de referência e **precisam ser ajustados** para o seu caso:

| Variável | Default atual | Observação |
|---|---|---|
| `environment_name` | `caos-demo-unifan` | Prefixo de todos os recursos; o escopo da policy IAM usa esse prefixo |
| `ecr_repository_name` | `unifan-sistemas-distribuidos-terraform` | Deve ser **igual** ao secret `ECR_REPOSITORY` |
| `domain_name` / `subdomain_name` | `rafaellevi.com.br` / `unifan-sistemas-distribuidos` | A **hosted zone precisa existir** na sua conta, senão o `apply` falha |
| `db_name` | `aula_pratica_db` | Nome do banco usado pela API |
| `log_group_name` | `/unifan/caos-demo-api` | Também usado nas queries do dashboard |

## Pré-requisitos

- Terraform >= 1.5 (testado com 1.9.8) e provider AWS >= 6.0.
- Credenciais AWS de **uma única conta** (as duas regiões usam a mesma conta).
- Um **bucket S3 para o state** (criado previamente; o Terraform não cria o próprio backend).
  Sem lock DynamoDB: evite dois `apply` simultâneos.
- A **hosted zone** do domínio configurado já existente no Route53 (é referenciada via
  `data source`; o Terraform cria apenas o subdomínio).

### Criar o bucket de state (uma vez)

```bash
aws s3api create-bucket --bucket SEU-BUCKET-DE-TFSTATE --region us-east-1
aws s3api put-bucket-versioning --bucket SEU-BUCKET-DE-TFSTATE \
  --versioning-configuration Status=Enabled
```

## Uso

O fluxo recomendado é pelo **GitHub Actions** (ver abaixo). Localmente, para quem tem credenciais:

```bash
export TF_STATE_BUCKET=SEU-BUCKET-DE-TFSTATE
./iac/scripts/apply.sh
```

Ou manualmente:

```bash
cd iac/terraform
cp backend.hcl.example backend.hcl            # ajuste o bucket
cp terraform.tfvars.example terraform.tfvars  # ajuste os valores
terraform init -backend-config=backend.hcl
terraform plan
terraform apply
```

`backend.hcl` e `terraform.tfvars` estão no `.gitignore`: **não os versione**.

### Redeploy da aplicação (nova imagem)

O pipeline builda a imagem, publica no ECR e dispara um *rolling instance refresh*:

```bash
export TF_STATE_BUCKET=SEU-BUCKET-DE-TFSTATE
./iac/scripts/refresh-instances.sh us-east-1
./iac/scripts/refresh-instances.sh eu-central-1
```

### Destruir tudo

```bash
export TF_STATE_BUCKET=SEU-BUCKET-DE-TFSTATE
./iac/scripts/destroy.sh
```

## CI/CD (GitHub Actions)

Secrets necessários (*Settings → Secrets and variables → Actions*):

| Secret | Descrição |
|--------|-----------|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Usuário IAM do pipeline |
| `TF_STATE_BUCKET` | Nome do bucket S3 onde o state é guardado |
| `ECR_REGISTRY` | `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com` |
| `ECR_REPOSITORY` | Nome do repositório ECR (igual a `ecr_repository_name`) |

Workflows:

- **Deploy Infrastructure** → roda **em paralelo** dois jobs: `terraform apply`
  (todas as regiões + dashboard + Route53) e o build/push da imagem. O job de build
  **não** usa `needs`; em vez disso aguarda ~20s no início (`wait-for-ecr: true`)
  para dar tempo do Terraform criar o ECR antes do `docker push`.
- **Terraform Plan** → `terraform plan` com resumo (criar/alterar/destruir). Não aplica nada.
- **Terraform CI** → em todo PR: `fmt`, `validate` e guarda contra segredos versionados.
- **Destroy Infrastructure** → `terraform destroy` (somente humano dispara).
- **Apply Database Schema** → lê `db_host`/`db_secret_name`/`rds_security_group_id`
  dos outputs do Terraform e aplica o schema. **Assume uma API Node.js** com `package.json` e
  o script `db:schema` (ver `.kiro/steering/40-contrato-da-api.md`).
- **Deploy Backend API** → build/push da imagem + instance refresh por região. Também roda a
  cada push na `main`.

### Permissões IAM do usuário do pipeline

O Terraform roda **como o usuário do pipeline** e chama as APIs da AWS diretamente. No `refresh`,
no `apply` e no `destroy` ele faz chamadas de leitura e escrita em IAM (`iam:ListRolePolicies`,
`iam:ListInstanceProfilesForRole`, `iam:TagInstanceProfile`...). Em vez de enumerar ação por ação,
conceda `iam:*` **restrito aos recursos da stack** (prefixo `<environment_name>-ec2-*`) mais
`iam:PassRole` com a condição `iam:PassedToService = ec2.amazonaws.com`.

O template pronto, com `ACCOUNT_ID` e `ENVIRONMENT_NAME` para preencher, está em
[`.kiro/skills/deploy-rest-api/assets/iam-pipeline-policy.template.json`](../../.kiro/skills/deploy-rest-api/assets/iam-pipeline-policy.template.json).

> **⚠️ A policy precisa ser aplicada manualmente ao usuário no console IAM (ou via CLI).**
> O JSON no repositório é só documentação; o Terraform não gerencia as permissões do próprio
> executor. Aguarde alguns segundos para propagar. Se o erro persistir com o mesmo `RequestID`,
> a policy ainda não foi aplicada de fato.

Diagnóstico de `AccessDenied`: `.kiro/steering/50-iam-e-permissoes.md`.
