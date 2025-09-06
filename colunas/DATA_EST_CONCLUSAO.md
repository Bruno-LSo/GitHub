
# DATA_EST_CONCLUSAO - Data Estimada de Conclusão

## Descrição Geral

A coluna `DATA_EST_CONCLUSAO` representa a data prevista para conclusão do protocolo, sendo extraída diretamente da tabela de protocolos e formatada para apresentação no padrão brasileiro.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_PROTOCOLO_CREANET`
- **Coluna de Origem**: `DT_EST_CONCLUSAO`
- **Tipo de Dado**: TIMESTAMP - Data e hora estimadas para conclusão

## Transformação e Formatação

### Processo de Conversão na Query

```sql
-- Na CTE PROTOCOLO_COMPLETO:
TO_CHAR(P.DT_EST_CONCLUSAO,'DD/MM/YYYY') AS DATA_EST_CONCLUSAO_FMT,

-- No SELECT final:
DATA_EST_CONCLUSAO_FMT AS DATA_EST_CONCLUSAO
```

### Exemplos Práticos dos Dados

Baseando-se nos exemplos reais da tabela `TB_PROTOCOLO_CREANET`:

| NR_PROT | DT_EST_CONCLUSAO (Original) | DATA_EST_CONCLUSAO (Formatada) |
|---------|------------------------------|--------------------------------|
| PR0309572023 | 2023-07-03 15:20:18.577143 | 03/07/2023 |
| PR0310732023 | 2023-07-03 23:11:22.275385 | 03/07/2023 |
| PR0311262023 | 2023-07-04 11:15:46.319637 | 04/07/2023 |
| PR0311962023 | 2023-07-04 14:48:03.487303 | 04/07/2023 |

## Utilização nas Regras de Negócio

### 1. Verificação de Protocolos Sem Pagamento

A `DATA_EST_CONCLUSAO` é utilizada para identificar protocolos vencidos sem pagamento em múltiplos pontos da query:

```sql
-- Na coluna CATEGORIA_STATUS:
WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
     AND DT_PAGAMENTO IS NULL 
     AND NR_BOLETO IS NOT NULL
THEN 'Sem pagamento'
```

**Exemplo Real**: 
- Protocolo PR0311262023 com DATA_EST_CONCLUSAO = 04/07/2023
- Se consultado após 04/07/2023 sem pagamento e com boleto emitido
- Resultado: CATEGORIA_STATUS = 'Sem pagamento'

### 2. Identificação na Coluna GRE

A mesma lógica aparece na determinação da GRE:

```sql
-- Na coluna GRE:
WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
     AND DT_PAGAMENTO IS NULL 
     AND NR_BOLETO IS NOT NULL
THEN 'Sem pagamento'
```

### 3. Classificação em GRE_SIMPLIFICADA

Para protocolos cancelados sem pagamento:

```sql
-- Na coluna GRE_SIMPLIFICADA:
WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
     AND DT_PAGAMENTO IS NULL 
     AND NR_BOLETO IS NOT NULL
THEN 'CANCELADO SEM PAGTO'
```

**Exemplo Real do Resultado da Query**:
| NR_PROT | DATA_EST_CONCLUSAO | NR_BOLETO | DT_PAGAMENTO | CATEGORIA_STATUS | GRE_SIMPLIFICADA |
|---------|-------------------|-----------|--------------|------------------|------------------|
| PR0311262023 | 04/07/2023 | 28027180231412766 | (null) | Sem pagamento | GRE10 |
| PR0311962023 | 04/07/2023 | 28027180231414609 | (null) | Sem pagamento | GRE10 |

## Conversão para Cálculos

A query utiliza `DAYS()` para converter a data em número de dias:

```sql
-- Conversão para comparação numérica:
DAYS(DT_EST_CONCLUSAO)
-- Comparado com:
DAYS(CURRENT DATE) AS DAYS_CURRENT
```

Esta conversão permite comparações aritméticas diretas para determinar se um protocolo ultrapassou o prazo estimado.

## Relacionamento com Outras Colunas

### NR_BOLETO e DT_PAGAMENTO
A `DATA_EST_CONCLUSAO` trabalha em conjunto com estas colunas para identificar:
- Protocolos com boleto emitido (`NR_BOLETO IS NOT NULL`)
- Sem pagamento realizado (`DT_PAGAMENTO IS NULL`)
- Após o prazo estimado (`DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO)`)

### Resultado na Query
Quando estas três condições são verdadeiras simultaneamente, o protocolo recebe classificações especiais:
- CATEGORIA_STATUS = 'Sem pagamento'
- GRE pode receber 'Sem pagamento'
- GRE_SIMPLIFICADA pode receber 'CANCELADO SEM PAGTO'

## Observações Técnicas

- A data é armazenada com precisão de timestamp mas apresentada apenas como DD/MM/YYYY
- A função `DAYS()` converte a data para um número inteiro representando dias desde uma data de referência do DB2
- Esta conversão é essencial para as comparações aritméticas de prazo na query
- O campo aparece em todas as linhas do resultado, independente do status do protocolo