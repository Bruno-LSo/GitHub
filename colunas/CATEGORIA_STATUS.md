# CATEGORIA_STATUS - Categorização por Prazo e Pagamento

## Descrição Geral

A coluna `CATEGORIA_STATUS` classifica os protocolos em categorias operacionais baseadas em prazo, pagamento e situação, facilitando a identificação de protocolos críticos e análise de conformidade com SLAs.

## Origem dos Dados

### Cálculo Baseado em Múltiplas Condições e Tabelas
- **Tipo de Dado**: VARCHAR - Categoria textual
- **Tabelas de Origem**:
  - `DB2INET.TB_STATUS_PROTOCOLO_CREANET`: DS_OBSERVACOES, DT_STATUS
  - `DB2INET.TB_PROTOCOLO_CREANET`: DT_ABERTURA, DT_EST_CONCLUSAO, FINISHED
  - `DB2INET.TB_BOLETO`: NR_BOLETO, DT_PAGAMENTO
  - `DB2INET.TB_TIPO_STATUS_PROTOCOLO`: DS_STATUS (via join)
  - `DB2INET.TB_SUBTIPO_PROTOCOLO`: Para SLA_OFICIAL

## Regras Completas de Categorização

```sql
-- No SELECT final:
CASE
    -- Regra 1: Cancelado por não pagamento
    WHEN DS_STATUS = 'Solicitação Cancelada' 
         AND FINISHED = 1 
         AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
    THEN 'Cancelado por não pagamento'
    
    -- Regra 2: Sem pagamento
    WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
         AND DT_PAGAMENTO IS NULL 
         AND NR_BOLETO IS NOT NULL
    THEN 'Sem pagamento'
    
    -- Regra 3: Comparação com SLA_OFICIAL
    WHEN (CASE
            WHEN DAYS_PAGAMENTO IS NOT NULL AND FINISHED = 1 
                THEN DAYS_STATUS - DAYS_PAGAMENTO
            WHEN DAYS_PAGAMENTO IS NOT NULL 
                THEN DAYS_CURRENT - DAYS_PAGAMENTO
            WHEN FINISHED = 1 
                THEN DAYS_STATUS - DAYS_ABERTURA
            ELSE DAYS_CURRENT - DAYS_ABERTURA
         END) <= SLA_OFICIAL
    THEN 'Dentro do Prazo'
    
    -- Regra 4: Else - Fora do Prazo
    ELSE 'Fora do Prazo'
END AS CATEGORIA_STATUS
```

## Detalhamento de Cada Categoria

### 1. Cancelado por não pagamento

```sql
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 'Cancelado por não pagamento'
```

**Condições necessárias**:
- Status = "Solicitação Cancelada"
- Protocolo finalizado (FINISHED = 1)
- Observações contêm "Boleto Vencido"

**Exemplo Real**:
| NR_PROT | DS_STATUS | FINISHED | DS_OBSERVACOES | NR_BOLETO | DT_PAGAMENTO | CATEGORIA_STATUS |
|---------|-----------|----------|----------------|-----------|--------------|------------------|
| PR0311262023 | Solicitação Cancelada | SIM | Boleto vencido: 28027180231412766 - Vencto.: 04/07/2023 00:00:00 | 28027180231412766 | NULL | Cancelado por não pagamento |
| PR0311962023 | Solicitação Cancelada | SIM | Boleto vencido: 28027180231414609 - Vencto.: 04/07/2023 00:00:00 | 28027180231414609 | NULL | Cancelado por não pagamento |

**Nota**: Estes protocolos também têm SLA = 0 devido à mesma regra.

### 2. Sem pagamento

```sql
WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
     AND DT_PAGAMENTO IS NULL 
     AND NR_BOLETO IS NOT NULL
THEN 'Sem pagamento'
```

**Condições necessárias**:
- Data atual > Data estimada de conclusão (protocolo vencido)
- Sem pagamento registrado (DT_PAGAMENTO IS NULL)
- Boleto emitido (NR_BOLETO IS NOT NULL)

**Lógica**: Identifica protocolos que ultrapassaram o prazo estimado e têm boleto emitido mas não pago.

**Exemplo Hipotético** (baseado na estrutura):
| NR_PROT | DATA_EST_CONCLUSAO | CURRENT_DATE | NR_BOLETO | DT_PAGAMENTO | CATEGORIA_STATUS |
|---------|-------------------|--------------|-----------|--------------|------------------|
| PRXXXXXX2023 | 04/07/2023 | 06/09/2025 | 28027180231414609 | NULL | Sem pagamento |

### 3. Dentro do Prazo

```sql
WHEN (CASE
        WHEN DAYS_PAGAMENTO IS NOT NULL AND FINISHED = 1 
            THEN DAYS_STATUS - DAYS_PAGAMENTO
        WHEN DAYS_PAGAMENTO IS NOT NULL 
            THEN DAYS_CURRENT - DAYS_PAGAMENTO
        WHEN FINISHED = 1 
            THEN DAYS_STATUS - DAYS_ABERTURA
        ELSE DAYS_CURRENT - DAYS_ABERTURA
     END) <= SLA_OFICIAL
THEN 'Dentro do Prazo'
```

**Lógica de Cálculo Interno**:

O CASE interno calcula o tempo decorrido usando EXATAMENTE a mesma lógica do SLA:

1. **Com pagamento e finalizado**: `DAYS_STATUS - DAYS_PAGAMENTO`
2. **Com pagamento em andamento**: `DAYS_CURRENT - DAYS_PAGAMENTO`
3. **Sem pagamento e finalizado**: `DAYS_STATUS - DAYS_ABERTURA`
4. **Sem pagamento em andamento**: `DAYS_CURRENT - DAYS_ABERTURA`

Este valor calculado é então comparado com SLA_OFICIAL:
- Se ≤ SLA_OFICIAL → "Dentro do Prazo"
- Se > SLA_OFICIAL → "Fora do Prazo" (cai no ELSE)

### 4. Fora do Prazo

```sql
ELSE 'Fora do Prazo'
```

**Condição**: Quando nenhuma das condições anteriores é atendida e o tempo calculado > SLA_OFICIAL.

**Exemplos Reais**:
| NR_PROT | SLA (calculado) | SLA_OFICIAL | CATEGORIA_STATUS | Análise |
|---------|-----------------|-------------|------------------|---------|
| PR0309572023 | 25 | 15 | Fora do Prazo | 25 > 15 |
| PR0310732023 | 27 | 15 | Fora do Prazo | 27 > 15 |

## Prioridade das Regras

A ordem de avaliação é CRUCIAL e segue esta hierarquia:

```
1º - Cancelado por não pagamento (tem prioridade máxima)
2º - Sem pagamento (protocolos vencidos com boleto não pago)
3º - Dentro do Prazo (comparação com SLA_OFICIAL)
4º - Fora do Prazo (ELSE - quando tempo > SLA_OFICIAL)
```

**Importante**: Um protocolo que atenda múltiplas condições receberá a categoria da PRIMEIRA regra verdadeira.

## Relação com Outras Colunas

### SLA vs CATEGORIA_STATUS

Note que CATEGORIA_STATUS NÃO usa diretamente o valor de SLA calculado, mas recalcula internamente:

| Coluna | Cálculo |
|--------|---------|
| **SLA** | Pode ser 0 para casos especiais (cancelado, empresa deferida) |
| **CATEGORIA_STATUS** | Sempre calcula o tempo real para comparar com SLA_OFICIAL |

**Exemplo de Diferença**:
| NR_PROT | SLA | Tempo Real | SLA_OFICIAL | CATEGORIA_STATUS |
|---------|-----|------------|-------------|------------------|
| PR0311262023 | 0 | 52 | 15 | Cancelado por não pagamento |

Aqui o SLA = 0 (devido ao cancelamento), mas CATEGORIA_STATUS ainda identifica como "Cancelado por não pagamento" pela regra específica.

### DS_STATUS_CLASSIFICACAO

Enquanto DS_STATUS_CLASSIFICACAO categoriza em "Fechado/Aberto/Exigência", CATEGORIA_STATUS foca em conformidade de prazo e situação de pagamento:

| DS_STATUS_CLASSIFICACAO | CATEGORIA_STATUS possíveis |
|------------------------|---------------------------|
| Fechado | Cancelado por não pagamento, Dentro do Prazo, Fora do Prazo |
| Aberto | Sem pagamento, Dentro do Prazo, Fora do Prazo |
| Exigência | Dentro do Prazo, Fora do Prazo |

## Casos Especiais

### Protocolos com SLA = 0

Protocolos que têm SLA = 0 devido a regras especiais ainda são categorizados:

```sql
-- Mesmo com SLA = 0, a comparação interna ainda ocorre
-- Exemplo: Cancelado com boleto vencido
SLA = 0 mas CATEGORIA_STATUS = 'Cancelado por não pagamento'
```

### Data Atual (CURRENT DATE)

A query usa `DAYS(CURRENT DATE)` calculado na CTE:

```sql
-- Na CTE PROTOCOLO_COMPLETO:
DAYS(CURRENT DATE) AS DAYS_CURRENT,
```

Conforme documentado no cabeçalho da query: "Current date is Saturday, September 06, 2025"

## Distribuição no Resultado Exemplo

Baseado nos dados fornecidos:

| CATEGORIA_STATUS | Quantidade | Protocolos |
|------------------|------------|------------|
| Fora do Prazo | 2 | PR0309572023, PR0310732023 |
| Cancelado por não pagamento | 2 | PR0311262023, PR0311962023 |
| Sem pagamento | 0 | (seria para PR0311262023/PR0311962023 mas "Cancelado" tem prioridade) |
| Dentro do Prazo | 0 | (nenhum no exemplo) |

## Observações Técnicas

- Categoria mutuamente exclusiva (apenas uma por protocolo)
- A ordem no CASE é fundamental - primeira condição verdadeira determina resultado
- "Cancelado por não pagamento" tem prioridade absoluta
- Recalcula internamente o tempo ao invés de usar SLA diretamente
- Essencial para dashboards de monitoramento e alertas
- Permite identificação rápida de protocolos críticos
- Base para relatórios de conformidade com SLAs regulamentares

---