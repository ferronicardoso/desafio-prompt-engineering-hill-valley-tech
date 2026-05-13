# Questão 03 - Relatório de redução de custos cloud

## Prompt

```markdown
# TASK
Analise o breakdown de custos AWS do arquivo CSV abaixo e identifique oportunidades de redução de custo alinhadas à meta de 15% de economia total, sem degradar SLA.

:::csv
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
:::

# Action
Para cada oportunidade, calcule:
1. economia mensal em USD
2. percentual de impacto na conta total
3. nível de esforço (baixo, médio, alto)
4. riscos e pré-requisitos técnicos. Priorize as oportunidades por impacto descendente.

# Goal
O objetivo é gerar um relatório que permita à liderança executiva (CEO + CTO) decidir rapidamente entre as opções mais viáveis e de maior impacto para atingir a meta de 15% de redução no próximo trimestre mantendo a confiabilidade dos sistemas.
```

## Modelo

Foi escolhido o `ChatGPT 5.5` pela sua capacidade de análise de dados estruturados e cálculos 

## Output

```markdown
Vou tratar o CSV como base fechada, calcular a conta total, a meta de 15% e priorizar cortes com premissas conservadoras para não mexer em SLA.

Conta mensal atual: **US$ 41.800**.  
Meta de 15%: **US$ 6.270/mês**.  
Plano abaixo soma **US$ 7.090/mês**, ou **16,96%**, com margem de segurança.

| Prioridade | Oportunidade | Economia mensal | Impacto total | Esforço     | Riscos / pré-requisitos |
| --- | --- | --- | --- | --- | --- |
| 1          | Reduzir EC2 on-demand via rightsizing, schedules e Savings Plans seletivo                     | **US$ 1.640**   | **3,92%**     | Médio       | Validar picos, autoscaling, janelas batch e workloads críticos. Não desligar capacidade sem baseline de CPU/memória/rede.         |
| 2          | CloudWatch Logs: reduzir retenção de 90 dias, filtrar logs ruidosos e mover histórico para S3 | **US$ 1.120**   | **2,68%**     | Baixo       | Garantir requisitos de auditoria/compliance. Manter logs críticos e incident response.                                            |
| 3          | EKS: consolidar nodes, ajustar requests/limits e autoscaling                                  | **US$ 1.005**   | **2,40%**     | Médio/Alto  | Exige análise por namespace, HPA/VPA, node groups e PDBs. Risco de throttling se requests forem agressivos.                       |
| 4          | RDS PostgreSQL: rightsizing controlado e otimização de storage/IO                             | **US$ 820**     | **1,96%**     | Médio       | Como é Multi-AZ, manter HA. Revisar CPU, conexões, IOPS, storage growth e janela de manutenção.                                   |
| 5          | S3 Standard: lifecycle para IA/Glacier e limpeza de objetos órfãos                            | **US$ 775**     | **1,85%**     | Baixo       | Mapear padrões de acesso. Evitar mover dados usados por aplicações online para classes com latência/custo de recuperação.         |
| 6          | Data Transfer Out: reduzir tráfego inter-região                                               | **US$ 475**     | **1,14%**     | Médio       | Exige mapear fluxos entre regiões, replicações, backups e chamadas entre serviços. Risco arquitetural se alterar rotas sem teste. |
| 7          | ElastiCache Redis: rightsizing e revisão de shards/réplicas                                   | **US$ 420**     | **1,00%**     | Médio       | Validar memória usada, evictions, hit ratio e failover. Não reduzir réplica crítica sem teste de resiliência.                     |
| 8          | NAT Gateway: reduzir gateways, usar VPC endpoints e revisar rotas                             | **US$ 360**     | **0,86%**     | Médio       | Cuidado com alta disponibilidade por AZ. VPC endpoints para S3/ECR/CloudWatch podem cortar tráfego sem piorar SLA.                |
| 9          | CloudWatch Metrics: reduzir métricas customizadas e cardinalidade                             | **US$ 180**     | **0,43%**     | Baixo       | Evitar remover métricas de SLO, capacidade e incidentes. Foco em métricas duplicadas ou de alta cardinalidade.                    |
| 10         | EBS gp3: remover volumes órfãos/snapshots antigos e ajustar provisionamento                   | **US$ 160**     | **0,38%**     | Baixo       | Confirmar volumes anexados, snapshots usados por rollback e backup policy.                                                        |
| 11         | Lambda: otimizar memória/duração e provisioned concurrency, se houver                         | **US$ 135**     | **0,32%**     | Baixo/Médio | Testar cold start, timeout e impacto em integrações assíncronas. | 

## Recomendação executiva

Para bater a meta no próximo trimestre sem degradar SLA, eu priorizaria este pacote:

**EC2 on-demand + CloudWatch Logs + EKS + RDS + S3 + Data Transfer + ElastiCache = US$ 6.255/mês**, praticamente a meta de **US$ 6.270/mês**.

Com **NAT Gateway** incluso, passa para **US$ 6.615/mês**, ou **15,83%**, criando folga.

## Decisão sugerida

Aprovar imediatamente as ações de **baixo risco**: CloudWatch Logs, S3 lifecycle, CloudWatch Metrics e EBS.  
Rodar em paralelo uma frente técnica para **EC2 on-demand, EKS e RDS**, porque aí está o grosso da economia, mas com maior necessidade de validação operacional.

```

## Justificativa

Foi aplicado o framework T-A-G definindo a tarefa de análise (Task), as operações específicas a realizar (Action: cálculos de economia, esforço, risco), e o objetivo empresarial (Goal: 15% de redução mantendo SLA). Essa estrutura garante que o modelo entregue análise quantitativa priorizada e acionável para tomada de decisão executiva.
