# Questão 06 - Módulo Terraform no padrão interno

## Prompt

```markdown
# Context
Você é um(a) especialista em Terraform e padrões corporativos de IaC.  
Precisa criar um **módulo Terraform reutilizável para S3** seguindo o padrão interno da empresa.

Regras obrigatórias do padrão:
1. Todo recurso deve ter as tags: `Owner`, `CostCenter`, `Environment`.
2. Todo nome de recurso deve usar prefixo `hvt-`.
3. Todo bucket S3 deve ter:
	- encryption habilitada (mínimo SSE-S3 / AES256),
	- versioning habilitado,
	- block public access total,
	- logging habilitado e configurado.
4. Todas as variáveis de entrada devem ficar em `variables.tf` com `description` e `type` obrigatórios.

Referência de estilo (seguir o mesmo padrão de organização):
- uso de `locals` para `common_tags`,
- `merge(local.common_tags, { Name = ... })` para tags finais,
- nomenclatura consistente com ambiente.

# Action
Gere o código completo de um módulo Terraform de S3 pronto para reuso por múltiplos times, incluindo:
1. `main.tf` com recursos necessários para atender todos os requisitos de segurança e compliance.
2. `variables.tf` com todas as variáveis de entrada necessárias, sempre com `description` e `type`.
3. `outputs.tf` com saídas úteis do módulo.
4. Exemplo de consumo do módulo (pasta `examples/` ou bloco de uso) mostrando chamada real.
5. Estrutura limpa, legível e pronta para produção.

Regras de implementação:
- Aplicar `locals.common_tags` com `Owner`, `CostCenter`, `Environment`.
- Garantir prefixo `hvt-` no nome do bucket e demais recursos auxiliares.
- Configurar criptografia no bucket (`AES256` no mínimo).
- Habilitar versionamento.
- Bloquear acesso público por completo.
- Configurar logging do bucket para um bucket de logs apropriado.
- Evitar placeholders vagos; entregar valores e nomes de variáveis coerentes.
- Comentar apenas quando necessário para clareza.

# Result
Retorne a resposta em seções separadas por arquivo, nesta ordem:
1. `main.tf`
2. `variables.tf`
3. `outputs.tf`
4. `examples/basic/main.tf` (ou equivalente)
5. `README.md` curto com instruções de uso

Cada seção deve conter código HCL válido e consistente entre si.

# Example
Exemplo de estilo esperado para tags e naming (adapte para S3):

```hcl
locals {
  common_tags = {
	Owner       = var.owner
	CostCenter  = var.cost_center
	Environment = var.environment
  }
}

tags = merge(local.common_tags, {
  Name = "hvt-s3-${var.environment}-${var.bucket_suffix}"
})

```

## Modelo

Foi utilizado o modelo `Claude Sonnet 4.6` pela expertise do Claude para com códigos e soluções técnica

## Output

```markdown
Rapha, direto ao ponto — aqui está o módulo completo, pronto para produção.

---

## `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  bucket_name     = "hvt-s3-${var.environment}-${var.bucket_suffix}"
  log_bucket_name = "hvt-s3-logs-${var.environment}-${var.bucket_suffix}"
}

# ─── Bucket de Logs ───────────────────────────────────────────────────────────

resource "aws_s3_bucket" "logs" {
  bucket        = local.log_bucket_name
  force_destroy = var.force_destroy

  tags = merge(local.common_tags, {
    Name    = local.log_bucket_name
    Purpose = "access-logs"
  })
}

resource "aws_s3_bucket_ownership_controls" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "logs" {
  depends_on = [aws_s3_bucket_ownership_controls.logs]
  bucket     = aws_s3_bucket.logs.id
  acl        = "log-delivery-write"
}

resource "aws_s3_bucket_public_access_block" "logs" {
  bucket = aws_s3_bucket.logs.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id

  versioning_configuration {
    status = "Enabled"
  }
}

# ─── Bucket Principal ─────────────────────────────────────────────────────────

resource "aws_s3_bucket" "main" {
  bucket        = local.bucket_name
  force_destroy = var.force_destroy

  tags = merge(local.common_tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = var.kms_key_arn != null ? "aws:kms" : "AES256"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = var.kms_key_arn != null ? true : false
  }
}

resource "aws_s3_bucket_logging" "main" {
  bucket        = aws_s3_bucket.main.id
  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "logs/${local.bucket_name}/"
}

resource "aws_s3_bucket_lifecycle_configuration" "main" {
  count  = length(var.lifecycle_rules) > 0 ? 1 : 0
  bucket = aws_s3_bucket.main.id

  dynamic "rule" {
    for_each = var.lifecycle_rules
    content {
      id     = rule.value.id
      status = rule.value.enabled ? "Enabled" : "Disabled"

      expiration {
        days = rule.value.expiration_days
      }

      noncurrent_version_expiration {
        noncurrent_days = rule.value.noncurrent_version_expiration_days
      }
    }
  }
}

---

## `variables.tf`

```hcl
variable "environment" {
  description = "Ambiente de implantação (ex: dev, staging, prod)."
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "O valor de 'environment' deve ser: dev, staging ou prod."
  }
}

variable "bucket_suffix" {
  description = "Sufixo único para diferenciar buckets dentro do mesmo ambiente (ex: artifacts, backups, uploads)."
  type        = string
}

variable "owner" {
  description = "Time ou pessoa responsável pelo recurso. Usado na tag 'Owner'."
  type        = string
}

variable "cost_center" {
  description = "Centro de custo para fins de billing. Usado na tag 'CostCenter'."
  type        = string
}

variable "kms_key_arn" {
  description = "ARN da chave KMS para criptografia SSE-KMS. Se nulo, usa SSE-S3 (AES256)."
  type        = string
  default     = null
}

variable "force_destroy" {
  description = "Permite destruir o bucket mesmo com objetos dentro. Use com cuidado em produção."
  type        = bool
  default     = false
}

variable "lifecycle_rules" {
  description = "Lista de regras de ciclo de vida para gerenciamento de objetos e versões."
  type = list(object({
    id                                = string
    enabled                           = bool
    expiration_days                   = number
    noncurrent_version_expiration_days = number
  }))
  default = []
}

---

## `outputs.tf`

```hcl
output "bucket_id" {
  description = "ID (nome) do bucket S3 principal."
  value       = aws_s3_bucket.main.id
}

output "bucket_arn" {
  description = "ARN do bucket S3 principal."
  value       = aws_s3_bucket.main.arn
}

output "bucket_domain_name" {
  description = "Nome de domínio regional do bucket principal."
  value       = aws_s3_bucket.main.bucket_regional_domain_name
}

output "log_bucket_id" {
  description = "ID (nome) do bucket de logs."
  value       = aws_s3_bucket.logs.id
}

output "log_bucket_arn" {
  description = "ARN do bucket de logs."
  value       = aws_s3_bucket.logs.arn
}

---

## `examples/basic/main.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}

module "s3_artifacts" {
  source = "../../"

  environment   = "prod"
  bucket_suffix = "artifacts"
  owner         = "platform-team"
  cost_center   = "CC-1042"
}

module "s3_backups" {
  source = "../../"

  environment   = "prod"
  bucket_suffix = "backups"
  owner         = "infra-team"
  cost_center   = "CC-2001"

  # SSE-KMS em vez de AES256
  kms_key_arn = "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"

  # Expira objetos após 90 dias; versões antigas após 30
  lifecycle_rules = [
    {
      id                                 = "expire-old-backups"
      enabled                            = true
      expiration_days                    = 90
      noncurrent_version_expiration_days = 30
    }
  ]
}

output "artifacts_bucket_arn" {
  value = module.s3_artifacts.bucket_arn
}

output "backups_bucket_arn" {
  value = module.s3_backups.bucket_arn
}

---

## `README.md`

```markdown
# Módulo Terraform — S3 Corporativo (`hvt-s3`)

Módulo reutilizável para provisionamento de buckets S3 em conformidade com o
padrão corporativo HVT: criptografia, versionamento, bloqueio de acesso público
e logging habilitados por padrão.

## Recursos criados

| Recurso | Descrição |
|---|---|
| `aws_s3_bucket.main` | Bucket principal |
| `aws_s3_bucket.logs` | Bucket dedicado a access logs |
| `aws_s3_bucket_public_access_block` | Bloqueio total de acesso público (ambos) |
| `aws_s3_bucket_versioning` | Versionamento habilitado (ambos) |
| `aws_s3_bucket_server_side_encryption_configuration` | SSE-S3 ou SSE-KMS |
| `aws_s3_bucket_logging` | Logs do bucket principal → bucket de logs |
| `aws_s3_bucket_lifecycle_configuration` | Opcional, via `lifecycle_rules` |

## Nomenclatura gerada

```text
hvt-s3-{environment}-{bucket_suffix}       # bucket principal
hvt-s3-logs-{environment}-{bucket_suffix}  # bucket de logs


## Uso mínimo

```hcl
module "s3_uploads" {
  source = "git::https://github.com/hvt-org/terraform-modules//s3?ref=v1.0.0"

  environment   = "prod"
  bucket_suffix = "uploads"
  owner         = "backend-team"
  cost_center   = "CC-3010"
}

## Variáveis

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `environment` | `string` | ✅ | `dev`, `staging` ou `prod` |
| `bucket_suffix` | `string` | ✅ | Identificador do propósito do bucket |
| `owner` | `string` | ✅ | Time responsável (tag Owner) |
| `cost_center` | `string` | ✅ | Centro de custo (tag CostCenter) |
| `kms_key_arn` | `string` | ❌ | ARN KMS para SSE-KMS; usa AES256 se nulo |
| `force_destroy` | `bool` | ❌ | Padrão: `false` |
| `lifecycle_rules` | `list(object)` | ❌ | Regras de expiração de objetos/versões |

## Outputs

| Nome | Descrição |
|---|---|
| `bucket_id` | Nome do bucket principal |
| `bucket_arn` | ARN do bucket principal |
| `bucket_domain_name` | Domínio regional do bucket principal |
| `log_bucket_id` | Nome do bucket de logs |
| `log_bucket_arn` | ARN do bucket de logs |

## Requisitos

- Terraform `>= 1.3`
- AWS Provider `>= 5.0`

---

**Pontos que valem destaque:**

- **Criptografia escalável** — começa em AES256 e sobe pra SSE-KMS com uma variável só; sem recriar recursos.
- **Bucket de logs isolado** — não mistura logs no bucket principal, evita loop de logging e facilita auditoria separada.
- **`lifecycle_rules` tipado** — usa `list(object(...))` com campos explícitos; sem `any`, sem surpresa em `terraform plan`.
- **`validation` em `environment`** — bloqueia valores inválidos antes mesmo do `apply`.
- **`force_destroy = false` por padrão** — proteção explícita contra destruição acidental em prod.
```

## Justificativa

 No Context fixamos o papel, cenário e regras obrigatórias de compliance, reduzindo ambiguidade, em Action, transformamos o pedido em entregáveis concretos (main.tf, variables.tf, outputs.tf, exemplo e README), guiando execução, já no Result, define-se formato de saída e ordem, aumentando consistência e facilidade de revisão, por fim, o Example ancora estilo/nomenclatura no padrão interno já existente, melhorando aderência prática ao que a empresa espera.
