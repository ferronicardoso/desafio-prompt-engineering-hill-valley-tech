# Questão 01 - Dockerfile para o Lift

## Prompt

```markdown
# Role
Você é um especialista em DevOps e containerização com foco em boas práticas de produção.

# Task
Sua tarefa é escrever um Dockerfile otimizado para a aplicação Lift, uma API Python/Flask, que será deployada em Kubernetes. A aplicação roda na porta 8080, usa gunicorn com 4 workers, e tem as dependências declaradas em requirements.txt. O runtime precisa expor as variáveis de ambiente DATABASE_URL e API_KEY.

O Dockerfile deve incluir:
1. imagem base Python enxuta e segura
2. usuário não-root, 
3. multi-stage build se aplicável para reduzir tamanho
4. cache eficiente de camadas, 
5. health check para Kubernetes
6. labels de metadados
7. sem secrets hardcoded
8. pronto para produção

# Format
Retorne apenas o Dockerfile comentado, sem explicações adicionais. Cada linha importante deve ter um comentário explicando o propósito.
```


## Modelo

Foi utilizado o modelo `Claude Sonnet 4.6` pela expertise do Claude para com códigos e soluções técnica

## Output

```dockerfile
# syntax=docker/dockerfile:1.4
# Habilita funcionalidades avançadas do BuildKit (cache mounts, heredocs, etc.)

# ─── STAGE 1: builder ────────────────────────────────────────────────────────
# Imagem slim para compilar dependências com extensões nativas (C bindings, etc.)
FROM python:3.13-slim AS builder

# Evita geração de .pyc e saída bufferizada — essencial em containers
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /build

# Copia apenas o manifesto primeiro — aproveita cache do Docker se não mudar
COPY requirements.txt .

# Instala dependências isoladas no diretório /install para copiar depois
# --no-cache-dir reduz tamanho; --prefix separa do sistema para o COPY seletivo
RUN pip install --no-cache-dir --upgrade pip \
 && pip install --no-cache-dir --prefix=/install -r requirements.txt

# ─── STAGE 2: runtime ────────────────────────────────────────────────────────
# Imagem mínima de produção — sem compiladores, sem ferramentas de build
FROM python:3.13-slim AS runtime

# Metadados OCI — rastreabilidade e governança de imagem
LABEL org.opencontainers.image.title="lift-api" \
      org.opencontainers.image.description="Lift API – Python/Flask via Gunicorn" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.vendor="Vitor Soluções" \
      org.opencontainers.image.base.name="python:3.13-slim"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    # Garante que o PATH inclua os binários instalados no stage builder
    PATH="/install/bin:$PATH" \
    PYTHONPATH="/install/lib/python3.13/site-packages"

# Variáveis obrigatórias declaradas sem valor — devem vir do ambiente externo
# (Kubernetes Secret, Helm values, etc.). Container falha rápido se ausentes.
ENV DATABASE_URL="" \
    API_KEY=""

# Cria grupo e usuário sem privilégios — princípio do menor privilégio
RUN groupadd --gid 1001 appgroup \
 && useradd --uid 1001 --gid appgroup --no-create-home --shell /bin/false appuser

WORKDIR /app

# Traz apenas as dependências compiladas do stage anterior — sem lixo de build
COPY --from=builder /install /install

# Copia código da aplicação com ownership direto — evita chown posterior caro
COPY --chown=appuser:appgroup . .

# Porta documentada — não publica automaticamente, apenas semântica + Kubernetes
EXPOSE 8080

# Troca para usuário sem privilégios antes de qualquer execução
USER appuser

# Health check compatível com Kubernetes readiness/liveness probe via HTTP
# interval: frequência da checagem | timeout: limite por tentativa
# start-period: grace period no boot | retries: falhas antes de UNHEALTHY
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"

# Gunicorn com 4 workers — ajuste workers conforme (2 * CPUs + 1) no Helm/env
# bind 0.0.0.0 necessário para tráfego externo dentro do pod
ENTRYPOINT ["gunicorn"]
CMD ["--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
```

## Justificativa

O uso do R-T-F é o mais indicado porque defini a expertise do modelo (Role), especificando a ação e contexto técnico do que o modelo deverá executr (Task), e defini de forma clara o formato esperado que no nosso caso é necessário um Dockerfile comentado. Essa estrutura garante que o modelo entregue código pronto para produção e bem documentado, sem desvios.