# Um dia na Hill Valley Tech

A Hill Valley Tech é uma empresa fictícia que serve de palco para este desafio. Tem cinco sistemas em produção, cada um com seu papel bem definido. O **Chronos** é o API gateway e a plataforma core, ponto de entrada de todo tráfego da empresa. Por trás dele, o **Ledger** é um data warehouse em PostgreSQL que guarda histórico de transações e eventos, enquanto o **Reactor** toca o processamento assíncrono por filas de mensagens. Em paralelo a tudo isso, o **Beacon** mantém a observabilidade do ambiente inteiro, métricas, logs e alertas, e é por ele que o plantão enxerga o que está acontecendo. Fora do core principal, o **Lift** é um produto em beta que o time vem amadurecendo à parte.

O time que toca essa operação também é enxuto. **Doc Brown**, o CTO, responde pela direção técnica. **Jennifer Parker** é a PM que prioriza o que o produto entrega. **Lorraine Baines** lidera a SRE e responde pelo plantão, é dela a cobrança por runbooks e procedimentos documentados. **George McFly** é o engenheiro sênior veterano que escreveu boa parte do sistema legado, e muita coisa ainda roda exatamente como ele deixou anos atrás. **Goldie Wilson**, a CEO, observa tudo pelo prisma de custo e crescimento. E **Strickland**, head de segurança e compliance, é quem bate carimbo nos padrões internos que todo código novo precisa seguir.

Nos próximos cenários você vai pegar algumas dessas demandas que chegam à mesa do time. Em cada questão, a entrega é um **prompt de IA** aplicando o framework indicado no enunciado, executado em um modelo, com o output registrado e a justificativa mostrando como os componentes do framework apareceram no prompt. A Questão 08 foge desse padrão: a escolha do framework fica por sua conta entre os cinco do capítulo, com comparação explícita contra duas alternativas.

---

## Questão 01 - Dockerfile para o Lift

O Lift vai sair das VMs onde vem rodando e entrar no cluster Kubernetes da empresa. O código já está pronto: uma API Python/Flask na porta 8080, dependências declaradas em `requirements.txt`, e duas variáveis de ambiente que precisam estar presentes no runtime, `DATABASE_URL` e `API_KEY`.

Estrutura do projeto:

```
lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py
```

Conteúdo de `requirements.txt`:

```
Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9
```

Em produção o serviço sobe com `gunicorn --bind 0.0.0.0:8080 --workers 4 app:app`.

Falta o Dockerfile. Seguir todas as boas práticas de criação.

**Tarefa.** Aplicando o framework **R-T-F**, escrever o prompt de IA que produza esse Dockerfile.

**Entregue.** Prompt, modelo, output e justificativa mostrando como Role, Task e Format aparecem no prompt.

---

## Questão 02 - Script de backup do Ledger

Lorraine chegou à conclusão de que o Ledger, o PostgreSQL que o George levantou na EC2 anos atrás, nunca teve rotina de backup automatizada. Hoje isso é uma dependência aberta no radar da SRE, e ela quer fechar com uma cron diária. O ambiente onde o script vai rodar:

- Host: `ledger-db.internal.hvt.io`
- Porta: `5432`
- Banco: `ledger_prod`
- Usuário de backup: `backup_user`
- Senha: variável de ambiente `PGPASSWORD`, populada pelo AWS Secrets Manager via IAM role da instância
- Região AWS: `us-east-1`
- SO da instância: Ubuntu 22.04 LTS
- Diretório de trabalho com 80 GB livres: `/var/backups/ledger`
- Tamanho médio atual do dump compactado: ~12 GB

O script precisa fazer o dump com `pg_dump`, compactar com `gzip`, subir o arquivo pro bucket S3 `hvt-ledger-backups` via `aws s3 cp`, manter 30 dias de retenção no S3 (removendo os mais antigos), registrar cada execução em `/var/log/ledger-backup.log` com timestamp, e sair com exit code adequado em caso de falha.

**Tarefa.** Aplicando o framework **R-T-F**, escrever o prompt de IA que produza esse script bash.

**Entregue.** Prompt, modelo, output e justificativa mostrando como Role, Task e Format aparecem no prompt.

---

## Questão 03 - Relatório de redução de custos cloud

Goldie apresentou a meta do próximo trimestre à diretoria: **15% de redução no custo cloud até o fim do período, sem degradar SLA**. Doc Brown pegou o breakdown de custos AWS do último mês e repassou a análise inicial para o time. O CSV está abaixo.

```
servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes
```

O relatório que vai voltar pra Goldie precisa trazer as oportunidades de economia priorizadas por impacto, quanto cada uma representa em percentual da conta total, o esforço de implementação (baixo, médio, alto) e os riscos ou pré-requisitos envolvidos em cada uma.

**Tarefa.** Aplicando o framework **T-A-G**, escrever o prompt de IA que, a partir do CSV acima, produza esse relatório alinhado à meta de 15%.

**Entregue.** Prompt, modelo, output e justificativa mostrando como Task, Action e Goal aparecem no prompt.

---

## Questão 04 - Relatório mensal de transações do Ledger

Jennifer está fechando a apresentação que vai levar pra Goldie na semana que vem, sobre crescimento de transações nos últimos 6 meses por categoria. Ela precisa dos números consolidados mas não escreve SQL, então mandou a demanda pra sua fila. O Ledger (PostgreSQL) tem o histórico completo, e as duas tabelas relevantes estão abaixo.

```
CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_category ON transactions(category);
```

```
CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Categorias em produção hoje: `subscription`, `one_time`, `refund` e `credit_adjustment`. Só entra no relatório quem tem `status = 'completed'`. O campo `amount_cents` está em centavos de real e precisa aparecer na saída em reais com 2 casas decimais. O recorte é dos últimos 6 meses corridos a partir de hoje (2026-04-24), agrupado por mês (no formato `YYYY-MM`) e por categoria, trazendo duas métricas por linha: quantidade de transações e volume total em reais. Ordenação final: mês crescente, depois categoria crescente.

**Tarefa.** Aplicando o framework **T-A-G**, escrever o prompt de IA que produza essa query SQL.

**Entregue.** Prompt, modelo, output e justificativa mostrando como Task, Action e Goal aparecem no prompt.

---

## Questão 05 - Modernizar deployment legado

Numa revisão de produção, Doc Brown puxou o manifest do Chronos e caiu neste deployment que o George escreveu três anos atrás. Desde então ninguém mexeu nele, e muita coisa que hoje é obrigatória no padrão da empresa ainda não está presente. Modernizar caiu na sua mesa.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"
```

A versão moderna precisa ter alta disponibilidade, imagem versionada (nada de `latest`), secrets fora do manifest, resource requests e limits, liveness e readiness probes, `securityContext` não-root e as demais práticas de produção que hoje são padrão na empresa.

**Tarefa.** Aplicando o framework **B-A-B**, escrever o prompt de IA que, recebendo esse manifest, produza a versão modernizada.

**Entregue.** Prompt, modelo, output e justificativa mostrando como Before, After e Bridge aparecem no prompt.

---

## Questão 06 - Módulo Terraform no padrão interno

Strickland, que fecha a ponta de segurança e compliance, publicou o padrão interno de IaC que todo módulo Terraform novo precisa seguir:

- Tags obrigatórias em todo recurso: `Owner`, `CostCenter`, `Environment`.
- Prefixo `hvt-` nos nomes de recursos.
- Todo bucket S3 com: encryption habilitada (SSE-S3 mínimo), versioning ativo, block public access total, logging configurado.
- Variáveis de entrada em `variables.tf` com `description` e `type` obrigatórios.

Doc Brown pediu um módulo Terraform reutilizável pra criar buckets S3 aderentes a esse padrão. O módulo vai ser consumido por todos os times da empresa, então precisa vir com exemplo de uso. Como referência de estilo, o módulo de VPC que já existe na empresa:

```
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```

**Tarefa.** Aplicando o framework **C-A-R-E**, escrever o prompt de IA que produza o módulo Terraform S3 aderente ao padrão, no mesmo estilo do exemplo.

**Entregue.** Prompt, modelo, output e justificativa mostrando como Context, Action, Result e Example aparecem no prompt.

---

## Questão 07 - Runbook para alerta recorrente

Toda semana, em média 4 vezes, o Beacon dispara o mesmo alerta no canal de plantão: `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`. Quem assume o plantão gasta de 30 a 40 minutos até resolver, e o tempo varia muito porque não existe procedimento documentado. Lorraine quer um runbook que qualquer plantonista consiga seguir de ponta a ponta sem depender de quem conhece o sistema. O ambiente que o runbook precisa considerar:

- Chronos roda no EKS, namespace `production`, 6 réplicas com HPA configurado (min 4, max 12, CPU target 70%).
- Deploy via Argo CD a partir do repositório `hvt/chronos-api`.
- Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS).
- Observabilidade: métricas expostas em `/metrics`, logs centralizados no Beacon, dashboards em Grafana.
- Ferramentas disponíveis para o plantão: `kubectl`, `aws cli`, `argocd cli`.
- Canal de plantão: `#oncall-chronos` no Slack.
- Time sênior de escalação: `@chronos-core` (SLA de resposta: 15 minutos em horário comercial, 30 fora).

O runbook precisa cobrir os passos iniciais de diagnóstico (com os comandos específicos a rodar), a verificação esperada ao final de cada passo, os critérios objetivos para escalar para o time sênior e o critério para encerrar o incidente.

**Tarefa.** Aplicando o framework **R-I-S-E**, escrever o prompt de IA que produza esse runbook procedural completo.

**Entregue.** Prompt, modelo, output e justificativa mostrando como Role, Input, Steps e Expectation aparecem no prompt.

---

## Questão 08 - Postmortem técnico de incidente em produção

Um incidente está em andamento durante pico de tráfego. Doc Brown entrou na call e precisa de um postmortem técnico em 20 minutos para decidir entre **rollback do deploy v2.48.0** (que subiu ontem) e **scaling emergencial** (aumento de limits do RDS e do pool de conexões). Os artefatos disponíveis para análise são os seguintes.

Evento do deploy anterior (ontem, 18:42 UTC):

```
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado endpoint POST /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s
```

Métricas do Beacon nos últimos 30 minutos:

```
timestamp                p99_latency_ms   req_rate_s   err_rate_pct
2026-04-24 13:30 UTC     420              1200         0.2
2026-04-24 13:45 UTC     510              1450         0.3
2026-04-24 14:00 UTC     780              1780         0.8
2026-04-24 14:10 UTC     2400             2100         4.5
2026-04-24 14:15 UTC     5200             2400         8.2
2026-04-24 14:20 UTC     8100             2650         11.7
```

Trecho do log do pod `chronos-api-79c4d8b9-xk2jp`:

```
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error
```

Estado do Reactor (fila `chronos-transactions`):

- 50.127 mensagens acumuladas, crescendo a ~800/min.
- Consumer lag atual: 18 minutos e aumentando.

Estado do cluster:

- Chronos: 12/12 pods running (HPA no máximo).
- CPU médio dos pods: 62%.
- Memória média dos pods: 71%.
- Conexões ativas ao Ledger: 240/250 (limite do RDS).

**Tarefa.** Escolher entre os cinco frameworks do capítulo (R-T-F, T-A-G, B-A-B, C-A-R-E ou R-I-S-E) aquele que se aplica melhor a esse cenário e escrever o prompt de IA que produza o postmortem técnico que o Doc pediu.

Nesta questão a **justificativa é o coração da entrega**. Além de explicar o framework escolhido e como seus componentes aparecem no prompt, comparar explicitamente com pelo menos **2 outros frameworks candidatos**, apontando o que se ganharia e o que se perderia em cada um.

**Entregue.** Prompt, modelo, output e justificativa estendida com comparação entre frameworks.

---

## Como a entrega deve ser feita

A entrega é um **repositório público no GitHub** contendo os prompts, outputs e justificativas das 8 questões. O link do repositório deve ser enviado ao final.

**A organização interna do repositório é livre e intencional.** No Capítulo 4 deste módulo será abordado Criação e Versionamento de Prompts, e a estrutura escolhida aqui será comparada com as práticas apresentadas lá. Guardar a decisão, ela é parte do aprendizado.

Cada questão exige 4 campos obrigatórios:

1. **Prompt**: o texto exato usado.
2. **Modelo**: qual modelo foi executado (ex.: GPT-4o, Claude Sonnet 4, Gemini 2.5 Pro, Llama 3 via Ollama) e, em 1 linha, por que esse modelo foi escolhido para a tarefa.
3. **Output**: a resposta real do modelo, na íntegra ou em trecho relevante.
4. **Justificativa**: em 2 a 4 linhas, mostrar como os componentes do framework indicado no enunciado aparecem no prompt. Na Q08 a justificativa precisa comparar o framework escolhido com 2 alternativas.

Algumas orientações práticas:

- Usar ao menos 2 providers distintos ao longo do desafio (OpenAI, Anthropic, Google, Meta ou local via Ollama).
- Registrar outputs ruins também. Se um resultado não ficou bom, comentar na justificativa o que faria diferente.
- Os dados dos cenários são fictícios, sem necessidade de sanitização.

## Sete das oito questões já trazem o framework definido no enunciado, ali o valor está em aplicar bem os componentes no prompt e explicar como cada um aparece. Na Q08 a escolha é sua entre os cinco do capítulo e a justificativa compara com 2 alternativas. Registrar o raciocínio, inclusive o que não funcionou, faz parte do valor da entrega.

## Bônus - Marketing pessoal (opcional)

Seção opcional, sem peso na avaliação. O que você aprendeu até este capítulo do curso é matéria-prima para posts de LinkedIn, threads no X ou artigos técnicos. Abaixo vão 3 temas baseados em aulas já cobertas (Fundamentos de IA, LLMs, Ferramentas e Plataformas, GenAI no cotidiano técnico, Fundamentos de Prompt Engineering), pensados para gerar autoridade na área.

### Tema 1 - Como escolher o modelo de IA certo: custo, latência, qualidade e privacidade

Aproveita o que foi visto sobre providers (OpenAI, Anthropic, Google, Meta via Ollama), formas de consumo (chatbot, agente, API) e os critérios para decidir entre eles. É a pergunta que todo time técnico está fazendo agora, e poucos profissionais sabem responder além de "uso ChatGPT".

**Formato sugerido:** carrossel de LinkedIn (5 a 7 slides). Hook (ex.: "Qual modelo de IA seu time deveria estar usando? GPT, Claude, Gemini ou Llama? A resposta depende de 4 critérios."), um slide por critério com exemplo prático, slide final com uma matriz de decisão simples.

### Tema 2 - Os 5 frameworks de prompt engineering aplicados a Cloud, DevOps e SRE

Aproveita as aulas dos frameworks (R-T-F, T-A-G, B-A-B, C-A-R-E, R-I-S-E) e do comparativo final. Um framework por seção, com um exemplo concreto da área (genérico, não os do desafio) e a indicação de quando usar cada um.

**Formato sugerido:** artigo longo no LinkedIn ou thread no X com 6 a 8 posts, um por framework e o último consolidando a árvore de decisão.

### Tema 3 - Tokens e janela de contexto: o que todo profissional técnico deveria entender antes de jogar prompt na IA

Aproveita as aulas de tokens, tokenização e janela de contexto. Tema técnico que pouca gente explora bem mas que impacta diretamente custo (pay-per-token) e qualidade da resposta (lost-in-the-middle, estouro da janela). Forte para diferenciar de quem só usa IA pela interface.

**Formato sugerido:** artigo técnico no Medium, Dev.to ou blog pessoal. Estrutura: o que é um token, por que importa para custo, por que importa para qualidade, exemplos práticos com prompts curtos vs prompts inflados.