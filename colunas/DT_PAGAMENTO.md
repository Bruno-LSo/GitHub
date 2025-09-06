# DT_PAGAMENTO - Data de Pagamento

## Descrição Geral

A coluna `DT_PAGAMENTO` apresenta a data em que o boleto associado ao protocolo foi efetivamente pago, sendo fundamental para o cálculo de SLA e determinação do status do protocolo.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_BOLETO`
- **Coluna de Origem**: `DT_PAGAMENTO`
- **Tipo de Dado**: DATE - Data do pagamento

### Obtenção através de JOIN

```sql
-- Na CTE PROTOCOLO_COMPLETO:
LEFT JOIN DB2INET.TB_BOLETO BOLETO ON BOLETO.CD_PROC = P.NR_PROT

-- Formatação:
TO_CHAR(BOLETO.DT_PAGAMENTO,'DD/MM/YYYY') AS DT_PAGAMENTO_FMT,
DAYS(BOLETO.DT_PAGAMENTO) AS DAYS_PAGAMENTO,

-- No SELECT final:
DT_PAGAMENTO_FMT AS DT_PAGAMENTO
```

## Utilização nas Regras de Negócio

### 1. Identificação de Cancelamento por Finalização Antes do Pagamento

```sql
-- Na coluna DS_STATUS:
WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO
    THEN 'Solicitação Cancelada'

-- Na coluna SLA:
WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO 
THEN 0
```

### 2. Cálculo do SLA com Base no Pagamento

```sql
-- Na coluna SLA:
WHEN DAYS_PAGAMENTO IS NOT NULL THEN
    CASE
        WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_PAGAMENTO
        ELSE DAYS_CURRENT - DAYS_PAGAMENTO
    END
```

### 3. Identificação de Protocolos Sem Pagamento

```sql
-- Em CATEGORIA_STATUS:
WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
     AND DT_PAGAMENTO IS NULL 
     AND NR_BOLETO IS NOT NULL
THEN 'Sem pagamento'
```

### 4. Verificação de Sem Distribuição

```sql
-- Na coluna GRE:
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NOT NULL
    AND COALESCE(DEPARTMENT, '') = ''
    AND COALESCE(GRE_DESC, '') = ''
    AND COALESCE(CIDADE, '') = ''
    AND COALESCE(ATRIBUIDO, '') = ''
THEN 'Sem distribuição'
```

## Exemplos Práticos no Resultado

Dados reais mostrando pagamentos:

| NR_PROT | NR_BOLETO | DT_PAGAMENTO | DS_STATUS | SLA |
|---------|-----------|--------------|-----------|-----|
| PR0309572023 | NULL | NULL | Solicitação Deferida | 25 |
| PR0310732023 | 28027180231410448 | 29/06/2023 | Solicitação Deferida | 27 |
| PR0311262023 | 28027180231412766 | NULL | Solicitação Cancelada | 52 |
| PR0311962023 | 28027180231414609 | NULL | Solicitação Cancelada | 52 |

## Impacto no Cálculo do SLA

### Com Pagamento (DT_PAGAMENTO NOT NULL)
- **Protocolo Finalizado**: SLA = DAYS_STATUS - DAYS_PAGAMENTO
- **Protocolo em Andamento**: SLA = DAYS_CURRENT - DAYS_PAGAMENTO

**Exemplo Real**:
PR0310732023: 
- DT_PAGAMENTO = 29/06/2023
- DT_STATUS = 26/07/2023
- SLA = 27 dias (contados a partir do pagamento)

### Sem Pagamento (DT_PAGAMENTO IS NULL)
- SLA calculado desde a abertura do protocolo
- Pode resultar em categoria "Sem pagamento" se vencido

## Relação com Outras Colunas

### DS_STATUS
Pagamento influencia transformação do status:
```sql
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NULL AND DS_STATUS = 'Solicitação Enviada'
    THEN 'Aguardando Pagamento'
```

### CATEGORIA_STATUS
Determina categorias baseadas em pagamento:
- "Sem pagamento": boleto emitido mas não pago após prazo
- "Cancelado por não pagamento": cancelado com boleto vencido

## Observações Técnicas

- Campo opcional (NULL quando não há pagamento)
- Convertido para formato DD/MM/YYYY para apresentação
- DAYS() usado para cálculos aritméticos
- Data crucial para início da contagem de SLA em serviços pagos
- Relacionamento 1:1 com protocolo através de CD_PROC

---
