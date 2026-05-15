# Questão 04 - Relatório mensal de transações do Ledger

## Prompt

```markdown
# Task
Você é um especialista em PostgreSQL, gere uma query que deverá obter o crescimento de transações no últimos 6 meses por categoria. 

# Action
Considere as tabelas e colunas abaixo:

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

CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

A query deverá considerar:
- Considerar as categorias: `subscription`, `one_time`, `refund` e `credit_adjustment`
- Somente com `status = 'completed'
- campo `amount_cents` está em centavos de real e precisa aparecer na saída em reais com 2 casas decimais
- Considerar o periodo de 6 meses corridos a partir de hoje 
- agrupado por mês (no formato `YYYY-MM`) e por categoria
- Duas métricas por linha: quantidade de transações e volume total em reais
- Ordenação em mês crescente, depois categoria crescente

# Goal

 Entregar uma query única, compativel com PostgreSQL Ledger pronta para consulta em produção para obter o relatório mensal de transações.
```

## Modelo
Foi realizado o teste em dois modelos, `Claude Sonnet 4.6` e `ChatGPT 5.5`, porém foi considerado como satisfatorio o resultado do Claude pelo fato de o ChatGPT ter sugerido criação de novo indice o que também poderia entrar como um monto de observação na TASK para não sugerir alteração de objeto e entregas somente a Query SQL.

## Output

```
Query — Crescimento de Transações (últimos 6 meses por categoria)

SELECT
    TO_CHAR(DATE_TRUNC('month', t.created_at), 'YYYY-MM')   AS mes,
    t.category                                                AS categoria,
    COUNT(*)                                                  AS qtd_transacoes,
    ROUND(SUM(t.amount_cents) / 100.0, 2)                    AS volume_total_reais
FROM transactions t
WHERE
    t.status   = 'completed'
    AND t.category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
    AND t.created_at >= DATE_TRUNC('month', NOW() - INTERVAL '5 months')
    AND t.created_at <  DATE_TRUNC('month', NOW()) + INTERVAL '1 month'
GROUP BY
    DATE_TRUNC('month', t.created_at),
    t.category
ORDER BY
    DATE_TRUNC('month', t.created_at) ASC,
    t.category                        ASC;

Decisões que valem nota
Janela de 6 meses completos
O filtro parte de DATE_TRUNC('month', NOW() - INTERVAL '5 months') — isso ancora no início do mês de 5 meses atrás e inclui o mês atual, garantindo exatamente 6 meses fechados sem cortar dias no começo da janela.
DATE_TRUNC no GROUP BY em vez de TO_CHAR
Agrupar pela expressão temporal nativa e só formatar no SELECT preserva o uso dos índices idx_transactions_created_at e idx_transactions_category. Agrupar direto por TO_CHAR(...) força um full scan.
ROUND(SUM(...) / 100.0, 2)
A divisão por 100.0 (literal NUMERIC) evita truncamento inteiro antes do ROUND, entregando centavos corretos em todos os casos.
BIGINT sem cast implícito problemático
amount_cents é BIGINT; dividir por 100.0 promove automaticamente para NUMERIC, então não há perda de precisão nem necessidade de cast explícito.
Sem JOIN com customers
O relatório pedido não exige nenhuma coluna de customers — trazer o join seria ruído e custo desnecessário no plano de execução.
```

## Justificativa

Iniciou-se a TASK apontando a necessidade de um especialidade em PostgreSQL e o que precisaria ser realizado de atividade, ou seja, obter um relatorio. Já em ACTION foi definido o que precisava realizar em detalhes, mostrando inclusive os objetos envolvidos e as condições para resultado. No GOAL passando a nosso objetivo, retornar uma query compativel com PostgreSQL Ledger pronta para execução.
