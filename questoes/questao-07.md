# Questão 07 - Runbook para alerta recorrente

## Prompt

```markdown
# Role
Você é um Site Reliability Engineer (SRE) sênior e Incident Commander experiente em Kubernetes/EKS, observabilidade e resposta a incidentes.
 
# Input
Contexto do incidente recorrente:
- Alerta: [CRITICAL] High memory usage on Chronos API pods (>85% for 10min)
- Frequência: ~4 vezes por semana
- Impacto atual: resolução leva 30–40 minutos e varia por ausência de procedimento
 
Ambiente:
- Serviço: Chronos API
- Plataforma: EKS
- Namespace: production
- Réplicas: 6
- HPA: min 4, max 12, CPU target 70%
- Deploy: Argo CD a partir de hvt/chronos-api
- Dependências: Ledger (PostgreSQL) e Reactor (SQS)
- Observabilidade: /metrics, logs no Beacon, dashboards no Grafana
- Ferramentas disponíveis: kubectl, aws cli, argocd cli
- Canal de plantão: #oncall-chronos (Slack)
- Escalação sênior: @chronos-core (SLA 15 min horário comercial, 30 min fora)

Requisitos obrigatórios do runbook:
1) Passos iniciais de diagnóstico com comandos específicos
2) Verificação esperada ao final de cada passo
3) Critérios objetivos para escalar ao time sênior
4) Critério claro para encerrar o incidente

# Steps
1. Gere um runbook procedural de ponta a ponta, em ordem cronológica, usando linguagem imperativa e direta.
2. Estruture cada etapa com os campos:
   - Objetivo
   - Comando(s)
   - Resultado esperado
   - Ação se falhar
3. Inclua comandos concretos de:
   - kubectl (pods, top, describe, events, logs)
   - aws cli (SQS/infra relevante)
   - argocd cli (status/sync/health da aplicação)
4. Diferencie claramente:
   - Diagnóstico inicial (primeiros 10 minutos)
   - Mitigação imediata
   - Investigação de causa provável
5. Defina critérios objetivos de escalação para @chronos-core, com gatilhos mensuráveis (tempo, severidade, sinais técnicos, impacto).
6. Defina critério de encerramento do incidente com condições verificáveis (estabilidade temporal, métricas, erro, fila, normalização dos pods).
7. Inclua checklist de comunicação no #oncall-chronos (abertura, atualizações, escalação, encerramento).
8. Entregue em Markdown com seções numeradas e checklist operacional.

# Expectation
Entregar um runbook completo, acionável e padronizado, que reduza variação entre plantonistas e permita execução sem dependência de especialistas. O documento deve ser autoexplicativo, conter comandos prontos para copiar/colar e critérios objetivos de decisão (escalar/encerrar) sem ambiguidades.
```

## Modelo

Foi utilizado o modelo `Claude Sonnet 4.6` pela expertise do Claude para com códigos e soluções técnica


## Output

```markdown
Agora vou criar o runbook diretamente em Markdown — documento completo, sem enrolação.Runbook completo entregue. Aqui o resumo do que foi construído e as decisões de design:

---

**Estrutura em 4 faixas temporais claras:**

- **0–10 min** → diagnóstico puro, sem tocar em nada. Seis passos sequenciais cobrindo pods, HPA, Argo CD, logs e eventos.
- **10–20 min** → mitigação cirúrgica: reinício seletivo antes de restart total, scale-out manual, verificação das duas dependências críticas (SQS e Postgres).
- **20–40 min** → investigação de causa raiz em paralelo, com correlação de Grafana, `/metrics`, Beacon e histórico de deploy.
- **Pós-incidente** → critérios objetivos de encerramento e comunicação padronizada.

**Decisões de design que eliminam variação entre plantonistas:**

- 8 gatilhos de escalação numerados (G1–G8) com condições mensuráveis — sem subjetividade do tipo "parece grave".
- 7 condições de encerramento verificáveis — ninguém fecha incidente antes de checar todas.
- Templates prontos para abertura, update a cada 10 min, escalação e encerramento.
- Checklist operacional linear no final — pode ser copiado para um ticket ou thread do Slack.

**O que este runbook ataca diretamente:**
O maior problema do incidente recorrente não é técnico — é a variação de 30–40 minutos por ausência de procedimento. Com este runbook, um plantonista sem contexto prévio do serviço consegue executar tudo até a mitigação dentro de 20 minutos, e só escala para o time sênior com evidências concretas em mãos.

# RUNBOOK — Chronos API: High Memory Usage (EKS)

> **Alerta:** `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`
> **Namespace:** `production` | **Réplicas:** 6 (HPA: 4–12) | **Deploy:** Argo CD → `hvt/chronos-api`
> **Dependências:** Ledger (PostgreSQL) · Reactor (SQS)
> **Canal de plantão:** `#oncall-chronos` | **Escalação:** `@chronos-core`

---

## Índice

1. [Comunicação — Abertura do Incidente](#1-comunicação--abertura-do-incidente)
2. [Diagnóstico Inicial (0–10 min)](#2-diagnóstico-inicial-010-min)
3. [Mitigação Imediata (10–20 min)](#3-mitigação-imediata-1020-min)
4. [Investigação de Causa Provável (20–40 min)](#4-investigação-de-causa-provável-2040-min)
5. [Critérios de Escalação para @chronos-core](#5-critérios-de-escalação-para-chronos-core)
6. [Critérios de Encerramento do Incidente](#6-critérios-de-encerramento-do-incidente)
7. [Comunicação — Atualizações e Encerramento](#7-comunicação--atualizações-e-encerramento)
8. [Checklist Operacional Completo](#8-checklist-operacional-completo)

---

## 1. Comunicação — Abertura do Incidente

**Assim que o alerta disparar**, poste imediatamente no `#oncall-chronos`:

```text
🔴 [INCIDENTE ABERTO] Chronos API — High Memory
- Horário: <HH:MM>
- Alerta: pods com uso de memória >85% por mais de 10 minutos
- Namespace: production
- Responsável: <seu @user>
- Status: iniciando diagnóstico

> ⚠️ **Não aguarde confirmação para começar o diagnóstico.** Poste o aviso e execute em paralelo.

---

## 2. Diagnóstico Inicial (0–10 min)

### 2.1 — Verificar estado geral dos pods

**Objetivo:** Identificar quais pods estão com problema e seu estado atual.

```bash
kubectl get pods -n production -l app=chronos-api \
  --sort-by='.status.startTime' \
  -o wide


**Resultado esperado:** Todos os pods em estado `Running`. Pods em `OOMKilled`, `Pending` ou `CrashLoopBackOff` indicam situação crítica.

**Ação se falhar:** Se 2 ou mais pods estiverem fora de `Running`, acione escalação imediata (ver Seção 5).

---

### 2.2 — Verificar consumo de memória em tempo real

**Objetivo:** Confirmar e quantificar o consumo atual de memória por pod.

```bash
kubectl top pods -n production -l app=chronos-api \
  --sort-by=memory


**Resultado esperado:** Uso de memória abaixo de 85% do limite configurado. Valores acima de 90% exigem mitigação imediata.

**Ação se falhar:** Se qualquer pod ultrapassar 90%, avance direto para a Seção 3.

---

### 2.3 — Inspecionar eventos recentes do namespace

**Objetivo:** Detectar OOMKill, evictions ou falhas de scheduling.

```bash
kubectl get events -n production \
  --sort-by='.lastTimestamp' \
  | grep -iE "chronos|OOMKill|Evict|BackOff|Failed" \
  | tail -30


**Resultado esperado:** Nenhum evento de `OOMKilling` ou `Evicted` nos últimos 15 minutos.

**Ação se falhar:** Registre o pod afetado e horário do evento. Inclua no post de atualização no canal.

---

### 2.4 — Verificar status do HPA

**Objetivo:** Confirmar se o autoscaler está respondendo ao aumento de carga.

```bash
kubectl get hpa -n production chronos-api -o wide
kubectl describe hpa -n production chronos-api


**Resultado esperado:** `REPLICAS` atual maior ou igual ao mínimo (4). Condição `AbleToScale: True` e `ScalingActive: True`.

**Ação se falhar:** Se o HPA estiver travado ou reportando `ScalingLimited`, verifique a disponibilidade de nodes com `kubectl get nodes` e registre.

---

### 2.5 — Verificar status do deploy via Argo CD

**Objetivo:** Confirmar que o estado atual em produção corresponde ao HEAD do repositório.

```bash
argocd app get chronos-api --refresh
argocd app list | grep chronos-api


**Resultado esperado:** `STATUS: Synced` e `HEALTH: Healthy`. Qualquer divergência é suspeita.

**Ação se falhar:** Se `OutOfSync`, não sincronize automaticamente sem investigar a causa. Registre e escale se necessário.

---

### 2.6 — Verificar logs dos pods com maior consumo

**Objetivo:** Identificar erros, stack traces ou padrões anômalos nos pods mais pesados.

```bash
# Substitua <pod-name> pelo pod com maior uso identificado no passo 2.2
kubectl logs <pod-name> -n production --tail=200 \
  | grep -iE "error|exception|warn|memory|heap|timeout|refused"


**Resultado esperado:** Sem stack traces de OutOfMemory, sem erros de conexão recorrentes com Ledger ou Reactor.

**Ação se falhar:** Capture os erros encontrados e inclua na investigação da Seção 4.

---

## 3. Mitigação Imediata (10–20 min)

> Execute esta seção se o uso de memória estiver acima de **90%** em qualquer pod ou se já houver pods `OOMKilled`.

### 3.1 — Forçar reinício dos pods mais pesados (rolling restart seletivo)

**Objetivo:** Liberar memória acumulada sem derrubar todo o deployment.

```bash
# Identifique os 2 pods com maior consumo (do passo 2.2) e delete-os
# O Deployment recriará automaticamente
kubectl delete pod <pod-name-1> <pod-name-2> -n production


**Resultado esperado:** Novos pods sobem em estado `Running` em até 60 segundos. Uso de memória dos novos pods abaixo de 60%.

**Ação se falhar:** Se os novos pods já subirem com uso alto (>70%), execute rolling restart completo:

```bash
kubectl rollout restart deployment/chronos-api -n production


---

### 3.2 — Forçar scale-out manual temporário

**Objetivo:** Distribuir a carga enquanto os pods antigos são reciclados.

```bash
kubectl scale deployment chronos-api -n production --replicas=10


**Resultado esperado:** 10 pods ativos em `Running` dentro de 2 minutos. Uso médio de memória cai proporcionalmente.

**Ação se falhar:** Se pods ficarem em `Pending`, há pressão de recursos nos nodes:

```bash
kubectl get nodes -o wide
kubectl describe nodes | grep -A5 "Allocated resources"


Se nodes estiverem saturados, escale para `@chronos-core` (Seção 5, gatilho G3).

---

### 3.3 — Verificar fila do Reactor (SQS)

**Objetivo:** Confirmar se há acúmulo anormal de mensagens causando pressão de processamento.

```bash
aws sqs get-queue-attributes \
  --queue-url <REACTOR_QUEUE_URL> \
  --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible \
  --region <AWS_REGION>


**Resultado esperado:** `ApproximateNumberOfMessages` abaixo do baseline normal (verifique no Grafana o valor típico). Valores 3x acima do normal indicam spike de ingestão.

**Ação se falhar:** Se a fila estiver com volume anormal, considere pausa temporária dos consumers via variável de ambiente ou redução de concorrência. Escale para `@chronos-core` antes de agir.

---

### 3.4 — Verificar conectividade com Ledger (PostgreSQL)

**Objetivo:** Descartar connection pool exausto ou queries longas como causa de retenção de memória.

```bash
kubectl exec -it <pod-name> -n production -- \
  sh -c "cat /proc/net/tcp | wc -l"


```bash
# Verificar número de conexões abertas no pod
kubectl exec -it <pod-name> -n production -- \
  sh -c "ss -tp | grep ESTABLISHED | wc -l"


**Resultado esperado:** Número de conexões dentro do pool configurado (geralmente ≤ 20 por pod). Valores acima indicam leak de conexão.

**Ação se falhar:** Registre o número e escale para `@chronos-core` se confirmar leak.

---

## 4. Investigação de Causa Provável (20–40 min)

> Esta seção é executada **em paralelo ou após a estabilização** para identificar a causa raiz.

### 4.1 — Analisar tendência de memória no Grafana

**Objetivo:** Identificar se o crescimento é gradual (leak) ou abrupto (spike de carga).

1. Acesse o dashboard `Chronos API — Memory & Resources` no Grafana
2. Filtre pelo namespace `production` e período das últimas 4 horas
3. Observe:
   - Crescimento linear contínuo → suspeita de **memory leak**
   - Spike repentino → suspeita de **carga pontual** (batch, integração externa, SQS)
   - Múltiplos pods com curva idêntica → problema sistêmico no código ou configuração

---

### 4.2 — Inspecionar métricas do endpoint /metrics

**Objetivo:** Coletar dados brutos de heap e GC expostos pela aplicação.

```bash
kubectl port-forward svc/chronos-api -n production 8080:8080 &

curl -s http://localhost:8080/metrics \
  | grep -iE "heap|gc|memory|thread|connection|queue"


**Resultado esperado:** Métricas de heap dentro dos limites. GC sem pausa longa (>500ms). Threads e connections dentro do esperado.

**Ação se falhar:** Capture o output completo e cole no canal `#oncall-chronos` como evidência para o time sênior.

---

### 4.3 — Verificar logs agregados no Beacon

**Objetivo:** Correlacionar o horário do spike com padrões específicos de requisição ou processamento.

1. Acesse o Beacon e aplique o filtro:
   - `service: chronos-api`
   - `level: error OR warn`
   - `timestamp: últimos 60 minutos`
2. Busque por padrões como:
   - Timeouts repetidos para Ledger ou Reactor
   - Erros de deserialização
   - Loops de retry sem backoff
   - Payloads grandes sendo carregados em memória

---

### 4.4 — Checar histórico de deploys recentes

**Objetivo:** Correlacionar o início do problema com uma mudança de código ou configuração.

```bash
argocd app history chronos-api


```bash
kubectl rollout history deployment/chronos-api -n production


**Resultado esperado:** Sem deploy novo nas últimas 2 horas antes do incidente.

**Ação se falhar:** Se houver deploy recente, considere rollback imediato após validar com `@chronos-core`:

```bash
argocd app rollback chronos-api <REVISION_NUMBER>


---

### 4.5 — Verificar configuração atual de limits e requests

**Objetivo:** Confirmar se os limites de memória foram alterados ou estão subdimensionados.

```bash
kubectl get deployment chronos-api -n production \
  -o jsonpath='{.spec.template.spec.containers[*].resources}' | jq .


**Resultado esperado:** `requests.memory` e `limits.memory` configurados explicitamente. Limits devem ser compatíveis com o workload normal (sem folga insuficiente).

---

## 5. Critérios de Escalação para @chronos-core

Escale imediatamente para `@chronos-core` via `#oncall-chronos` se **qualquer** dos gatilhos abaixo for atingido:

| ID | Gatilho | Condição Mensurável |
|----|---------|-------------------|
| G1 | **Pods em OOMKill contínuo** | 2 ou mais pods reiniciados por OOMKill em menos de 15 minutos |
| G2 | **Uso de memória não cede** | Memória permanece acima de 90% após rolling restart e scale-out |
| G3 | **Nodes saturados** | Pods ficam em `Pending` por mais de 5 minutos por falta de recursos |
| G4 | **Serviço degradado** | Taxa de erros 5xx acima de 5% por mais de 5 minutos |
| G5 | **Deploy suspeito** | Deploy novo identificado nas 2h anteriores ao incidente |
| G6 | **Fila SQS anormal** | Volume 3x acima do baseline por mais de 10 minutos |
| G7 | **Tempo esgotado** | 30 minutos de incidente sem estabilização |
| G8 | **Leak confirmado** | Métricas de heap mostram crescimento linear sem platô após restart |

**SLA de resposta:**
- Horário comercial (08h–18h): **15 minutos**
- Fora do horário comercial: **30 minutos**

**Template de escalação no canal:**

```text
⚠️ [ESCALAÇÃO] @chronos-core necessário
- Gatilho: <G1 / G2 / ...>
- Evidência: <descrição objetiva + comando executado + output>
- Ações já executadas: <lista>
- Horário do alerta original: <HH:MM>
- Tempo total de incidente: <X min>


---

## 6. Critérios de Encerramento do Incidente

O incidente só pode ser encerrado quando **todas** as condições abaixo forem verificadas:

| # | Condição | Como Verificar |
|---|----------|---------------|
| 1 | Uso de memória abaixo de **70%** em todos os pods | `kubectl top pods -n production -l app=chronos-api` |
| 2 | Estável por no mínimo **15 minutos consecutivos** | Observar no Grafana sem novo spike |
| 3 | Nenhum pod em `OOMKilled`, `CrashLoopBackOff` ou `Pending` | `kubectl get pods -n production -l app=chronos-api` |
| 4 | Taxa de erros 5xx abaixo de **1%** | Dashboard Grafana → `Chronos API — Error Rate` |
| 5 | Fila SQS normalizada | `aws sqs get-queue-attributes ...` — valor no baseline |
| 6 | HPA estável | `kubectl get hpa chronos-api -n production` — sem flapping |
| 7 | Réplicas reduzidas ao normal (mínimo 4) | `kubectl scale` reverter para réplicas padrão se scale manual foi feito |

```bash
# Reverter scale manual após estabilização
kubectl scale deployment chronos-api -n production --replicas=6


---

## 7. Comunicação — Atualizações e Encerramento

### Atualizações durante o incidente (a cada 10 minutos)

```text
🟡 [UPDATE] Chronos API — High Memory | T+10min
- Status: <em mitigação / investigando causa / estabilizando>
- Ação atual: <o que está sendo feito>
- Impacto: <há degradação visível? taxa de erro?>
- Próximo update: <HH:MM>


### Mensagem de encerramento

```text
✅ [INCIDENTE ENCERRADO] Chronos API — High Memory
- Horário de abertura: <HH:MM>
- Horário de encerramento: <HH:MM>
- Duração total: <X min>
- Causa provável: <memory leak / spike de carga / deploy / SQS / desconhecida>
- Ações executadas: <lista resumida>
- Causa raiz confirmada: sim / não (investigação pendente)
- Próximos passos: <criar ticket, ajustar limits, revisar código, etc.>


> Se a causa raiz **não foi confirmada**, abra um ticket de follow-up antes de encerrar.

---

## 8. Checklist Operacional Completo

Use este checklist durante a execução. Marque cada item conforme avança.

### Abertura
- [ ] Alerta recebido e confirmado
- [ ] Mensagem de abertura postada em `#oncall-chronos`
- [ ] Cronômetro iniciado

### Diagnóstico Inicial (0–10 min)
- [ ] 2.1 — Estado dos pods verificado
- [ ] 2.2 — Consumo de memória medido com `kubectl top`
- [ ] 2.3 — Eventos do namespace analisados
- [ ] 2.4 — HPA verificado e funcional
- [ ] 2.5 — Status Argo CD verificado (Synced + Healthy)
- [ ] 2.6 — Logs dos pods com maior uso analisados

### Mitigação Imediata (10–20 min)
- [ ] 3.1 — Pods mais pesados reiniciados (se >90%)
- [ ] 3.2 — Scale-out manual aplicado (se necessário)
- [ ] 3.3 — Fila SQS verificada
- [ ] 3.4 — Conectividade com Ledger verificada

### Investigação (20–40 min)
- [ ] 4.1 — Tendência de memória analisada no Grafana
- [ ] 4.2 — Métricas do `/metrics` coletadas
- [ ] 4.3 — Logs do Beacon correlacionados
- [ ] 4.4 — Histórico de deploys verificado
- [ ] 4.5 — Limits e requests de memória inspecionados

### Escalação
- [ ] Gatilho de escalação identificado (ou descartado)
- [ ] `@chronos-core` acionado (se aplicável) com evidências

### Encerramento
- [ ] Todas as 7 condições de encerramento verificadas
- [ ] Réplicas revertidas ao padrão (se scale manual foi feito)
- [ ] Mensagem de encerramento postada em `#oncall-chronos`
- [ ] Ticket de causa raiz aberto (se não confirmada)

---

*Versão: 1.0 — Revisão recomendada após cada incidente maior.*
```


## Justificativa

No Role definimos o perfil e expertise (SRE/Incident Commander) e nível de responsabilidade.
Já no Input fornecemos todo o contexto operacional real (infra, ferramentas, dependências, SLAs e requisitos).
Em Steps, é dados as devidas orientações e a forma de construção do runbook, incluindo estrutura por etapa e comandos obrigatórios.
Por fim, em Expectation, fixamos qualidade, formato e critérios de sucesso do resultado (acionável, padronizado, objetivo e sem ambiguidade).

