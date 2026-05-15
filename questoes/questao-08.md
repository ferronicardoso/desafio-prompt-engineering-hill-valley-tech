# Questão 08 - Postmortem técnico de incidente em produção

## Prompt

```markdown
# Role
Você é um SRE Sênior com expertise em sistemas de pagamento, PostgreSQL e arquitetura de microsserviços. Está na call de guerra com o CTO e precisa entregar um postmortem técnico em 20 minutos.

# Input
Analise os artefatos abaixo — changelog do deploy v2.48.0, métricas do Beacon (30 min), logs do pod chronos-api-79c4d8b9-xk2jp, estado do Reactor e estado do cluster — e produza um postmortem técnico executável que embase a decisão entre:
A) Rollback imediato para v2.47.0
B) Scaling emergencial (aumento de limits do RDS + pool de conexões)

Artefatos:

Evento do deploy anterior (ontem, 18:42 UTC):

```text
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado endpoint POST /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s

Métricas do Beacon nos últimos 30 minutos:

```text
timestamp                p99_latency_ms   req_rate_s   err_rate_pct
2026-04-24 13:30 UTC     420              1200         0.2
2026-04-24 13:45 UTC     510              1450         0.3
2026-04-24 14:00 UTC     780              1780         0.8
2026-04-24 14:10 UTC     2400             2100         4.5
2026-04-24 14:15 UTC     5200             2400         8.2
2026-04-24 14:20 UTC     8100             2650         11.7


Trecho do log do pod `chronos-api-79c4d8b9-xk2jp`:

```text
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error


Estado do Reactor (fila `chronos-transactions`):

- 50.127 mensagens acumuladas, crescendo a ~800/min.
- Consumer lag atual: 18 minutos e aumentando.

Estado do cluster:

- Chronos: 12/12 pods running (HPA no máximo).
- CPU médio dos pods: 62%.
- Memória média dos pods: 71%.
- Conexões ativas ao Ledger: 240/250 (limite do RDS).


# Steps
1. Identifique a causa-raiz provável a partir dos logs e changelog (correlacione eventos, não apenas liste-os).
2. Avalie o impacto atual: usuários afetados, degradação de SLA, risco de cascata.
3. Confronte as duas opções com os dados disponíveis — tempo de recuperação estimado, riscos residuais, reversibilidade.
4. Emita uma recomendação clara com nível de confiança (alto/médio/baixo) e justificativa em 2–3 linhas.
5. Liste as 3 ações imediatas de contenção, independente da decisão tomada.
6. Aponte os itens de follow-up pós-incidente (máximo 5, priorizados).

# Expectation
- Output em seções numeradas seguindo os Steps acima.
- Linguagem técnica direta, sem floreios — é uma call de guerra.
- Recomendação final destacada em negrito, com nível de confiança explícito.
- Tempo de leitura máximo: 3 minutos.
- Sem disclaimers genéricos como "pode ser necessário investigar mais".
```

## Modelo

Foi utilizado o modelo `Claude Sonnet 4.6` pela expertise do Claude para com códigos e soluções técnica

## Output

```markdown
# Postmortem Técnico — Incidente chronos-api v2.48.0
**14:20 UTC | Call de Guerra**

---

## 1. Causa-Raiz

O incidente tem gatilho único no deploy de ontem às 18:42 UTC, com manifestação tardia por acúmulo de carga.

**Cadeia de causalidade:**

```text
psycopg 3.1.18 → 3.2.0
    └─ Mudança de comportamento no pool (biblioteca interna refatorada)
         └─ pool.max = 20 por pod × 12 pods = 240 conexões ativas
              └─ Limite RDS: 250 → headroom de 10 conexões apenas
                   └─ Timeout reduzido 5s→2s + queries lentas = conexões presas
                        └─ Pool exhausted (waiting=147)
                             └─ Circuit breaker OPEN (87%)
                                  └─ Reactor sem consumidor funcional → lag 18min


**Smoking gun:** O pool de conexões foi movido para uma nova biblioteca interna sem validar o comportamento do psycopg 3.2.0 em cenário de conexões longas. O timeout agressivo de 2s não libera conexão — segura e expira com erro, mantendo o slot ocupado até rollback da transação no driver. Com HPA já no teto, não há mais pods para diluir o problema.

---

## 2. Impacto Atual

| Dimensão | Estado |
|---|---|
| Erro rate | 11,7% e subindo (~3% a cada 5 min) |
| p99 latência | 8.100ms — SLA estourado (assumindo <1000ms) |
| Fila Reactor | 50k msgs, +800/min, lag 18min → **risco de perda de eventos** |
| Conexões RDS | 240/250 — **margem crítica, qualquer spike trava o banco** |
| HPA | Maxed out — scaling horizontal não resolve mais |
| Usuários afetados | Todo fluxo que passa por `/v2/transactions/batch` e dependentes |

**Risco de cascata iminente:** Reactor acumulando sem consumidor ativo. Se o consumer lag ultrapassar o retention do broker, há perda de mensagens não processadas — isso é dado financeiro.

---

## 3. Confronto das Opções

### Opção A — Rollback para v2.47.0

| | |
|---|---|
| **Tempo de recuperação** | 4–8 min (Argo CD sync + rolling restart) |
| **Risco residual** | Baixo — estado conhecido e estável |
| **Reversibilidade** | Total |
| **Fila Reactor** | Começa a drenar em ~10 min após rollback |
| **Contra** | Perde o endpoint `/v2/transactions/batch` até novo deploy corrigido |

### Opção B — Scaling emergencial (RDS limits + pool)

| | |
|---|---|
| **Tempo de recuperação** | 15–30 min (modificar parameter group RDS + reiniciar pool) |
| **Risco residual** | Alto — causa-raiz no driver não é tratada, apenas aliviada |
| **Reversibilidade** | Parcial — RDS scaling pode exigir reboot dependendo da instância |
| **Fila Reactor** | Continua crescendo durante a intervenção |
| **Contra** | Se o comportamento do psycopg 3.2.0 for o problema real, aumentar conexões apenas posterga o colapso |

---

## 4. Recomendação

> **Execute o Rollback imediato para v2.47.0 (Opção A).**
>
> **Confiança: ALTO.**

O pool exhausted não é problema de capacidade — é problema de vazamento de conexão introduzido pela combinação `psycopg 3.2.0 + nova biblioteca + timeout 2s`. Aumentar RDS limits com driver quebrado é colocar mais água num balde furado. Rollback resolve a causa-raiz em menos de 8 minutos, devolve headroom ao RDS e desbloqueia o Reactor antes que o lag vire perda de dados.

---

## 5. Ações Imediatas de Contenção (independente da decisão)

1. **Agora:** Pinnar manualmente o circuit breaker do ledger-client em HALF-OPEN nos pods ainda funcionais — reduz erro rate enquanto a decisão é executada.
2. **Agora:** Pausar o consumer do Reactor na fila `chronos-transactions` — congela o lag onde está e evita processamento duplicado/corrompido durante a instabilidade.
3. **Junto com o rollback/scaling:** Forçar encerramento das 147 conexões em waiting no pool via `pg_terminate_backend` no RDS — libera os slots presos antes do novo deploy estabilizar.

---

## 6. Follow-up Pós-Incidente (priorizados)

1. **[P0 — hoje]** Reproduzir o comportamento do psycopg 3.2.0 em ambiente de staging com carga simulada antes de qualquer redeploy da v2.48.0.
2. **[P0 — hoje]** Auditar o comportamento da nova biblioteca interna de pool: verificar se `connection_timeout` e `max_waiting` estão explicitamente configurados ou herdando defaults do driver.
3. **[P1 — amanhã]** Revisar o timeout de 2s no Ledger — com queries em tabela `transactions` sem índice confirmado na query logada, 2s é inviável em carga real.
4. **[P1 — essa sprint]** Adicionar alerta em `connections_active > 80% do limite RDS` no Beacon — esse incidente foi visível 40 minutos antes do colapso nos dados de métricas.
5. **[P2 — essa sprint]** Definir política de smoke test obrigatório pós-deploy para endpoints novos (`/v2/transactions/batch`) com carga progressiva antes de expor ao tráfego total.
```

## Justificativa

O R-I-S-E é o único framework dos cinco que sequencia o raciocínio antes de definir o output. O bloco Steps funciona como chain-of-thought explícito: força o modelo a passar por correlação causal → impacto → trade-off → recomendação, nessa ordem. Em diagnósticos de incidente, pular etapas ou inverter a ordem produz postmortems superficiais ("pool cheio aumentar limite"). O Expectation fecha o cerco: limita ambiguidade no formato e proíbe outputs vagos.


| Framework | O que ganharia | O que perderia |
|---|---|---|
| **R-T-F** | Prompt mais curto | Sem controle do raciocínio intermediário — output vira narrativa, não diagnóstico |
| **C-A-R-E** | Consistência de formato se houver template organizacional | Depende de exemplo real; sem ele, polui o contexto e aumenta risco de alucinação |
| **T-A-G** | Extremamente enxuto | Raso demais para análise comparativa de opções |
| **B-A-B** | Bom para transformação de estado | Projetado para redesign, não para diagnóstico em tempo real |

**Linha final:** R-I-S-E é o único que controla processo *e* produto — os outros entregam o formato, mas não disciplinam o caminho até ele.