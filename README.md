# consulta-revenue
Esta query SQL gera um relatório detalhado do faturamento mensal, discriminado por filiais (SBC/Barueri, Extrema, Serra), e inclui uma linha de totalização geral. É ideal para análises de performance e acompanhamento financeiro.


# Análise da Query SQL - Faturamento por Filial

## 1. Comando Principal: `SELECT` com `UNION ALL`

Esta query utiliza uma estrutura mais complexa, combinando dois comandos `SELECT` através de `UNION ALL`. O primeiro `SELECT` extrai dados mensais de faturamento por filial, enquanto o segundo adiciona uma linha de totais gerais. O resultado final é ordenado para apresentar os dados de forma cronológica e lógica.

## 2. Estrutura Geral da Query

A query é composta por:
1. **SELECT externo**: Seleciona as colunas finais da subquery
2. **Subquery com UNION ALL**: Combina dados mensais com linha de total
3. **ORDER BY**: Ordena os resultados por ordem lógica (dados mensais primeiro, total depois) e cronológica

## 3. Primeira Parte da UNION: Dados Mensais

### Colunas e Transformações:

**Tradução de Mês com `CASE`:**
- `CASE date_format(purchased_at, '%m') WHEN '01' THEN 'Janeiro' ... END AS Mes`
- **O que faz:** Extrai o mês da data de compra usando `date_format(purchased_at, '%m')` e traduz o número do mês (01-12) para nomes em português (Janeiro-Dezembro).
- **Tipo de dado esperado:** String.

**Extração de Ano:**
- `date_format(purchased_at, '%Y') AS Ano`
- **O que faz:** Extrai o ano da data de compra e o mantém como string.
- **Tipo de dado esperado:** String.

**Cálculos de Faturamento por Filial com Formatação:**
- `regexp_replace(format('%.2f', SUM(CASE WHEN filial = 'sbc' OR filial = 'barueri' THEN total ELSE 0 END)), '(\d)(?=(\d{3})+\.)', '$1') AS SBC`
- `regexp_replace(format('%.2f', SUM(CASE WHEN filial = 'extrema' THEN total ELSE 0 END)), '(\d)(?=(\d{3})+\.)', '$1') AS Extrema`
- `regexp_replace(format('%.2f', SUM(CASE WHEN filial = 'serra' THEN total ELSE 0 END)), '(\d)(?=(\d{3})+\.)', '$1') AS Serra`
- `regexp_replace(format('%.2f', SUM(total)), '(\d)(?=(\d{3})+\.)', '$1') AS Total_RS`

**O que fazem:** Estas linhas realizam uma operação complexa em três etapas:
1. **`SUM(CASE WHEN filial = 'X' THEN total ELSE 0 END)`**: Soma os valores de `total` apenas para pedidos da filial específica.
2. **`format('%.2f', ...)`**: Formata o número com duas casas decimais.
3. **`regexp_replace(..., '(\d)(?=(\d{3})+\.)', '$1')`**: Aplica formatação de milhares (adiciona separadores de milhares).

**Tipo de dado esperado:** Numérico na origem, String formatada no resultado.

**Observação:** A filial SBC inclui tanto 'sbc' quanto 'barueri' (`filial = 'sbc' OR filial = 'barueri'`).

**Campo de Ordenação:**
- `1 as ordem`
- **O que faz:** Adiciona um campo constante para controlar a ordenação na query externa.

### Cláusulas FROM e WHERE:

**FROM:**
- `FROM datalake_gold.orders_channel`
- **O que faz:** Especifica a tabela de origem, que parece ser uma tabela consolidada de pedidos em um data lake.

**WHERE:**
- `WHERE purchased_at >= TIMESTAMP '2024-07-01 00:00:00' AND purchased_at <= CURRENT_TIMESTAMP`
- `and authorized_at is not null`
- `and cancelled_at is null`

**O que fazem:**
- Filtra pedidos a partir de 1º de julho de 2024 até o momento atual.
- Inclui apenas pedidos autorizados (`authorized_at is not null`).
- Exclui pedidos cancelados (`cancelled_at is null`).

**GROUP BY:**
- `GROUP BY date_format(purchased_at, '%m'), date_format(purchased_at, '%Y')`
- **O que faz:** Agrupa os dados por mês e ano, permitindo que as funções `SUM` calculem totais mensais.

## 4. Segunda Parte da UNION: Linha de Total

A segunda parte do `UNION ALL` cria uma linha de totais gerais:

**Campos Fixos:**
- `'Total Faturamento' AS Mes` e `'' AS Ano`
- **O que fazem:** Criam uma linha identificadora para os totais gerais.

**Cálculos de Total Geral:**
- Utiliza as mesmas fórmulas de soma e formatação da primeira parte, mas sem `GROUP BY`, resultando em totais gerais para todo o período.

**Campo de Ordenação:**
- `2 as ordem`
- **O que faz:** Garante que a linha de total apareça após os dados mensais na ordenação final.

## 5. Cláusula ORDER BY Externa

```sql
ORDER BY
    ordem,
    Ano,
    CASE Mes
        WHEN 'Janeiro' THEN 1
        WHEN 'Fevereiro' THEN 2
        ...
        WHEN 'Dezembro' THEN 12
        ELSE 99
    END
```

**O que faz:** Ordena os resultados em três níveis:
1. **`ordem`**: Dados mensais (1) aparecem antes do total (2).
2. **`Ano`**: Ordenação cronológica por ano.
3. **`CASE Mes`**: Ordenação cronológica por mês, convertendo nomes de meses para números.

## 6. O que a Query Faz no Geral?

Esta query gera um relatório de faturamento mensal por filial, incluindo:
- Dados mensais de faturamento para cada filial (SBC/Barueri, Extrema, Serra).
- Total mensal geral.
- Linha de totais gerais para todo o período.
- Formatação de valores com separadores de milhares.
- Ordenação cronológica dos dados.

O resultado é um relatório estruturado que permite análise comparativa de performance entre filiais ao longo do tempo, com totais consolidados.

## 7. É SQL? MySQL? PostgreSQL e etc?

Sim, é SQL. Esta query utiliza funcionalidades que são **específicas do MySQL** ou sistemas compatíveis.

### Compatibilidade Detalhada:

#### 1. MySQL
- **Compatibilidade:** **Totalmente compatível**. Todas as funções utilizadas são nativas do MySQL:
  - `date_format()`: Formatação de datas específica do MySQL.
  - `format()`: Formatação numérica específica do MySQL.
  - `regexp_replace()`: Função de expressão regular do MySQL.
  - `CURRENT_TIMESTAMP`: Padrão SQL, mas amplamente suportado.

#### 2. PostgreSQL
- **Compatibilidade:** **Não é diretamente compatível**. Requer adaptações:
  - `date_format()` → `TO_CHAR()`
  - `format('%.2f', number)` → `TO_CHAR(number, 'FM999999999.00')`
  - `regexp_replace()` → `REGEXP_REPLACE()` (sintaxe ligeiramente diferente)

#### 3. SQL Server
- **Compatibilidade:** **Não é diretamente compatível**. Requer adaptações:
  - `date_format()` → `FORMAT()` ou `DATENAME()`/`DATEPART()`
  - `format('%.2f', number)` → `FORMAT(number, 'N2')`
  - `regexp_replace()` → Não existe nativamente (SQL Server 2022+ tem algumas funções de regex)

#### 4. Oracle
- **Compatibilidade:** **Não é diretamente compatível**. Requer adaptações:
  - `date_format()` → `TO_CHAR()`
  - `format()` → `TO_CHAR()` com máscaras específicas
  - `regexp_replace()` → `REGEXP_REPLACE()` (sintaxe pode diferir)

**Conclusão:** A query é escrita em SQL com **dialeto específico do MySQL** devido ao uso intensivo de funções de formatação e manipulação de strings/datas específicas do MySQL. Para ser executada em outros SGBDs, praticamente todas as funções de formatação precisariam ser reescritas usando as funções nativas de cada sistema.
