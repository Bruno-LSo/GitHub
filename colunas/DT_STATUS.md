# DT_STATUS - Data do Status

## Descrição Geral

A coluna `DT_STATUS` apresenta a data do último status registrado para o protocolo, representando a data da última movimentação ou atualização significativa no processo de atendimento.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_STATUS_PROTOCOLO_CREANET`
- **Coluna de Origem**: `DT_STATUS`
- **Tipo de Dado**: TIMESTAMP - Data e hora do registro do status

### Obtenção através da CTE

```sql
-- Na CTE ULTIMO_STATUS para obter o status mais recente:
SELECT 
    SP.CD_PROT,
    SP.DT_STATUS,
    -- outras colunas...
    ROW_NUMBER() OVER (PARTITION BY SP.CD_PROT ORDER BY SP.CD_STAT_PROT DESC) AS RN
FROM DB2INET.TB_STATUS_PROTOCOLO_CREANET SP

-- Na CTE PROTOCOLO_COMPLETO:
TO_CHAR(US.DT_STATUS,'DD/MM/YYYY') AS DT_STATUS_FMT,
DAYS(US.DT_STATUS) AS DAYS_STATUS,

-- No SELECT final:
DT_STATUS_FMT AS DT_STATUS
```

## Transformações Aplicadas

### Formatação para Apresentação
```sql
TO_CHAR(US.DT_STATUS,'DD/MM/YYYY') AS DT_STATUS_FMT
```
Converte o TIMESTAMP para formato brasileiro DD/MM/YYYY

### Conversão para Cálculos
```sql
DAYS(US.DT_STATUS) AS DAYS_STATUS
```
Converte para número de dias para cálculos de SLA

## Utilização nas Regras de Negócio

### 1. Verificação de Cancelamento Antes do Pagamento

```sql
-- Na coluna DS_STATUS:
WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO
    THEN 'Solicitação Cancelada'
```

**Exemplo**: Se DAYS_STATUS (data do último status) < DAYS_PAGAMENTO (data de pagamento), indica que o protocolo foi finalizado antes do pagamento ser processado.

### 2. Cálculo do SLA para Protocolos Finalizados

```sql
-- Na coluna SLA:
WHEN DAYS_PAGAMENTO IS NOT NULL THEN
    CASE
        WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_PAGAMENTO
        ELSE DAYS_CURRENT - DAYS_PAGAMENTO
    END
ELSE
    CASE
        WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_ABERTURA
        ELSE DAYS_CURRENT - DAYS_ABERTURA
    END
```

## Exemplos Práticos no Resultado

Dados reais da query mostrando DT_STATUS:

| NR_PROT | DT_STATUS | DS_STATUS | ATRIBUIDO | FINALIZADO |
|---------|-----------|-----------|-----------|------------|
| PR0309572023 | 21/07/2023 | Solicitação Deferida | rosana.torre4031 | SIM |
| PR0310732023 | 26/07/2023 | Solicitação Deferida | rosana.torre4031 | SIM |
| PR0311262023 | 18/08/2023 | Solicitação Cancelada | lucas.rodrigues4435 | SIM |
| PR0311962023 | 18/08/2023 | Solicitação Cancelada | lucas.rodrigues4435 | SIM |

## Relação com Data de Finalização

A `DT_STATUS` torna-se a data de finalização quando o status é conclusivo:

```sql
-- Na coluna DT_FINALIZACAO:
CASE
    WHEN DS_STATUS IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida')
    THEN DT_STATUS_FMT
    ELSE NULL
END AS DT_FINALIZACAO
```

## Comparações Temporais

### Com Data de Pagamento
```sql
DAYS_STATUS < DAYS_PAGAMENTO
```
Identifica protocolos finalizados antes do pagamento

### Com Data de Abertura
```sql
DAYS_STATUS - DAYS_ABERTURA
```
Calcula o tempo total de processamento para protocolos finalizados

## Observações Técnicas

- Sempre representa o último status através do filtro `RN = 1` na CTE
- A precisão de TIMESTAMP é mantida internamente mas apresentada como DD/MM/YYYY
- Fundamental para cálculos de SLA e tempo de atendimento
- A função `DAYS()` permite comparações aritméticas diretas entre datas
- Em protocolos finalizados, representa efetivamente a data de conclusão

---
