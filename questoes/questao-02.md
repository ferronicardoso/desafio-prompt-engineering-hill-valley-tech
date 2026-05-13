# Questão 02 - Script de backup do Ledger

## Prompt

```markdown
# Role
Você é um especialista em shell scripting e operações de banco de dados PostgreSQL em ambientes AWS.

# Task
Sua tarefa é escrever um script bash para automação de backup do banco de dados Ledger (PostgreSQL) que será executado em cron diária. O script deve se conectar a ledger-db.internal.hvt.io:5432, banco ledger_prod, com usuário backup_user. A senha vem da variável de ambiente PGPASSWORD (populada pelo AWS Secrets Manager via IAM role). O dump deve ser compactado com gzip, enviado para o bucket S3 hvt-ledger-backups na região us-east-1 via aws s3 cp, manter os últimos 30 dias de retenção (deletando backups mais antigos), registrar cada execução em /var/log/ledger-backup.log com timestamp, e retornar exit codes apropriados para sucesso (0) e falha (1). O script roda em Ubuntu 22.04 LTS, com /var/backups/ledger disponível (80 GB livres) e tamanho médio do dump compactado é ~12 GB.

# Format
Retorne apenas o script bash production-ready, com tratamento robusto de erros, validações de permissão e espaço em disco, e comentários em pontos críticos. Sem explicações adicionais.

```

## Modelo

Foi utilizado o modelo `Claude Sonnet 4.6` pela expertise do Claude para com códigos e soluções técnica

## Output

```bash
#!/usr/bin/env bash
# =============================================================================
# ledger-backup.sh — Backup automatizado do PostgreSQL Ledger para S3
# Cron sugerido: 0 2 * * * /opt/scripts/ledger-backup.sh
# =============================================================================

set -euo pipefail

# -----------------------------------------------------------------------------
# Constantes
# -----------------------------------------------------------------------------
readonly DB_HOST="ledger-db.internal.hvt.io"
readonly DB_PORT="5432"
readonly DB_NAME="ledger_prod"
readonly DB_USER="backup_user"
readonly S3_BUCKET="s3://hvt-ledger-backups"
readonly S3_REGION="us-east-1"
readonly BACKUP_DIR="/var/backups/ledger"
readonly LOG_FILE="/var/log/ledger-backup.log"
readonly RETENTION_DAYS=30
readonly MIN_DISK_GB=20       # Mínimo de espaço livre exigido antes de iniciar
readonly EXPECTED_SIZE_GB=12  # Tamanho estimado do dump compactado

readonly TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
readonly DUMP_FILENAME="ledger_prod_${TIMESTAMP}.dump.gz"
readonly DUMP_PATH="${BACKUP_DIR}/${DUMP_FILENAME}"
readonly S3_KEY="${S3_BUCKET}/${DUMP_FILENAME}"

# -----------------------------------------------------------------------------
# Logging
# -----------------------------------------------------------------------------
log() {
    local level="$1"
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

log_info()  { log "INFO " "$@"; }
log_warn()  { log "WARN " "$@"; }
log_error() { log "ERROR" "$@"; }

# -----------------------------------------------------------------------------
# Cleanup — garante remoção do dump local em qualquer saída
# -----------------------------------------------------------------------------
cleanup() {
    local exit_code=$?
    if [[ -f "${DUMP_PATH}" ]]; then
        log_info "Removendo dump local temporário: ${DUMP_PATH}"
        rm -f "${DUMP_PATH}"
    fi
    if [[ ${exit_code} -ne 0 ]]; then
        log_error "Script encerrado com falha (exit code: ${exit_code})"
    fi
    exit "${exit_code}"
}
trap cleanup EXIT

# -----------------------------------------------------------------------------
# Função de falha explícita
# -----------------------------------------------------------------------------
die() {
    log_error "$*"
    exit 1
}

# -----------------------------------------------------------------------------
# Validações de ambiente
# -----------------------------------------------------------------------------
validate_environment() {
    log_info "Iniciando validações de ambiente..."

    # Dependências obrigatórias
    local deps=("pg_dump" "gzip" "aws" "df" "du")
    for dep in "${deps[@]}"; do
        command -v "${dep}" &>/dev/null || die "Dependência ausente: '${dep}'. Instale antes de continuar."
    done

    # Variável de senha via Secrets Manager → IAM role
    [[ -n "${PGPASSWORD:-}" ]] || die "Variável PGPASSWORD não definida. Verifique a IAM role e o Secrets Manager."

    # Diretório de backup
    [[ -d "${BACKUP_DIR}" ]] || die "Diretório de backup não existe: ${BACKUP_DIR}"

    # Permissão de escrita no diretório de backup
    [[ -w "${BACKUP_DIR}" ]] || die "Sem permissão de escrita em: ${BACKUP_DIR}"

    # Permissão de escrita no log
    local log_dir
    log_dir=$(dirname "${LOG_FILE}")
    [[ -w "${log_dir}" ]] || die "Sem permissão de escrita no diretório de log: ${log_dir}"

    # Espaço em disco: exige MIN_DISK_GB livres + buffer do dump esperado
    local required_gb=$(( MIN_DISK_GB + EXPECTED_SIZE_GB ))
    local available_kb
    available_kb=$(df --output=avail "${BACKUP_DIR}" | tail -1)
    local available_gb=$(( available_kb / 1024 / 1024 ))

    if [[ ${available_gb} -lt ${required_gb} ]]; then
        die "Espaço insuficiente em ${BACKUP_DIR}: ${available_gb} GB disponíveis, mínimo exigido: ${required_gb} GB."
    fi

    log_info "Espaço disponível em ${BACKUP_DIR}: ${available_gb} GB — OK"

    # Conectividade com o banco (pg_isready não exige senha)
    if ! pg_isready -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" -t 10 &>/dev/null; then
        die "Banco de dados inacessível em ${DB_HOST}:${DB_PORT}. Verifique conectividade e credenciais."
    fi
    log_info "Conectividade com PostgreSQL confirmada."

    # Credenciais AWS — valida que a IAM role está ativa
    if ! aws sts get-caller-identity --region "${S3_REGION}" &>/dev/null; then
        die "Credenciais AWS inválidas ou IAM role sem permissão. Verifique a role da instância EC2."
    fi
    log_info "Credenciais AWS validadas."
}

# -----------------------------------------------------------------------------
# Dump do banco com pg_dump → gzip direto (sem arquivo intermediário não-compactado)
# -----------------------------------------------------------------------------
perform_dump() {
    log_info "Iniciando dump: ${DB_NAME}@${DB_HOST}:${DB_PORT} → ${DUMP_PATH}"

    # --no-password: usa PGPASSWORD; -Fc seria ideal, mas gzip externo mantém
    # compatibilidade com pg_restore e inspeção direta via zcat/zgrep
    if ! pg_dump \
            --host="${DB_HOST}" \
            --port="${DB_PORT}" \
            --username="${DB_USER}" \
            --dbname="${DB_NAME}" \
            --no-password \
            --format=plain \
            --verbose \
            2>>"${LOG_FILE}" \
        | gzip -9 > "${DUMP_PATH}"; then
        die "Falha durante pg_dump | gzip. Verifique o log: ${LOG_FILE}"
    fi

    # Verifica se o arquivo foi criado e tem conteúdo real
    [[ -s "${DUMP_PATH}" ]] || die "Dump gerado está vazio: ${DUMP_PATH}"

    local dump_size_mb
    dump_size_mb=$(du -m "${DUMP_PATH}" | cut -f1)
    log_info "Dump concluído: ${DUMP_PATH} (${dump_size_mb} MB)"
}

# -----------------------------------------------------------------------------
# Upload para S3
# -----------------------------------------------------------------------------
upload_to_s3() {
    log_info "Enviando dump para S3: ${S3_KEY}"

    if ! aws s3 cp "${DUMP_PATH}" "${S3_KEY}" \
            --region "${S3_REGION}" \
            --storage-class STANDARD_IA \
            --only-show-errors; then
        die "Falha no upload para S3: ${S3_KEY}"
    fi

    log_info "Upload concluído: ${S3_KEY}"
}

# -----------------------------------------------------------------------------
# Retenção: remove backups no S3 com mais de RETENTION_DAYS dias
# -----------------------------------------------------------------------------
apply_retention() {
    log_info "Aplicando política de retenção: últimos ${RETENTION_DAYS} dias em ${S3_BUCKET}"

    local cutoff_date
    cutoff_date=$(date -d "-${RETENTION_DAYS} days" +"%Y-%m-%dT%H:%M:%S")

    # Lista objetos e filtra os anteriores ao cutoff
    local old_keys
    old_keys=$(aws s3api list-objects-v2 \
                    --bucket "$(basename "${S3_BUCKET}")" \
                    --query "Contents[?LastModified<='${cutoff_date}'].Key" \
                    --region "${S3_REGION}" \
                    --output text 2>>"${LOG_FILE}" || true)

    if [[ -z "${old_keys}" || "${old_keys}" == "None" ]]; then
        log_info "Nenhum backup antigo a remover."
        return 0
    fi

    local deleted_count=0
    while IFS= read -r key; do
        [[ -z "${key}" ]] && continue
        if aws s3 rm "${S3_BUCKET}/${key}" --region "${S3_REGION}" &>>"${LOG_FILE}"; then
            log_info "Removido: s3://$(basename "${S3_BUCKET}")/${key}"
            (( deleted_count++ )) || true
        else
            log_warn "Falha ao remover: ${key} — continuando..."
        fi
    done <<< "${old_keys}"

    log_info "Retenção concluída: ${deleted_count} backup(s) removido(s)."
}

# -----------------------------------------------------------------------------
# Ponto de entrada
# -----------------------------------------------------------------------------
main() {
    log_info "========================================================"
    log_info "Início do backup — Ledger DB → S3"
    log_info "Script PID: $$"

    validate_environment
    perform_dump
    upload_to_s3
    apply_retention

    log_info "Backup finalizado com sucesso: ${DUMP_FILENAME}"
    log_info "========================================================"
    exit 0
}

main "$@"
```

## Justificativa

O prompt aplica R-T-F definindo expertise em shell e PostgreSQL (Role), detalha a ação completa incluindo requisitos técnicos, integrações
AWS, retenção de dados e logging (Task), e especifica o formato esperado: script production-ready com error handling e sem prolixidade
(Format). Essa estrutura garante entrega imediata de código executável em cron.

Uma observação importante, ao executar a primeira vez, o `Claude Sonnet 4.6` entregou justificativas e outras recomendações antes de entregar o script e inclusive entregou sugestão de `cron`.
 