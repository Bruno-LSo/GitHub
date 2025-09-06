# SLA - Tempo de Atendimento

## Descrição Geral

A coluna `SLA` calcula o tempo efetivo de atendimento do protocolo em dias, considerando diferentes cenários de pagamento, finalização e cancelamento.

## Origem dos Dados

### Cálculo Dinâmico
- **Tipo de Dado**: INTEGER - Número de dias
- **Cálculo**: Baseado em datas de abertura, status, pagamento e situação atual

## Regras de Cálculo

A query implementa um CASE complexo para determinar o SLA:

```sql
-- No SELECT final:
CASE
    -- Regra 1: Cancelado por não pagamento
    WHEN DS_STATUS = 'Solicitação Cancelada' 
         AND FINISHED = 1 
         AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
    THEN 0
    
    -- Regra 2: Finalizado antes do pagamento
    WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO 
    THEN 0
    
    -- Regra 3: Empresa com Registro deferido
    WHEN NM_TP_PROT = 'Empresa' AND NM_SUBTP_PROT = 'Registro de Empresa' 
         AND DS_STATUS = 'Solicitação Deferida' 
    THEN 0
    
    -- Regra 4: Cálculo padrão com pagamento
    WHEN DAYS_PAGAMENTO IS NOT NULL THEN
        CASE
            WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_PAGAMENTO
            ELSE DAYS_CURRENT - DAYS_PAGAMENTO
        END
    
    -- Regra 5: Cálculo padrão sem pagamento
    ELSE
        CASE
            WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_ABERTURA
            ELSE DAYS_CURRENT - DAYS_ABERTURA
        END
END AS SLA
```

## Cenários de Cálculo

### 1. SLA = 0 (Casos Especiais)

**Cancelamento por Boleto Vencido**:
```sql
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 0
```

**Exemplo Real**:
| NR_PROT | DS_STATUS | DS_OBSERVACOES | SLA |
|---------|-----------|----------------|-----|
| PR0311262023 | Solicitação Cancelada | Boleto vencido: 28027180231412766... | 0 |

### 2. SLA com Pagamento

**Protocolo Finalizado**:
```sql
WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_PAGAMENTO
```

**Exemplo Real**:
| NR_PROT | DT_PAGAMENTO | DT_STATUS | FINISHED | SLA |
|---------|--------------|-----------|----------|-----|
| PR0310732023 | 29/06/2023 | 26/07/2023 | SIM | 27 |

Cálculo: DAYS(26/07/2023) - DAYS(29/06/2023) = 27 dias

### 3. SLA sem Pagamento

**Protocolo em Andamento**:
```sql
WHEN FINISHED = 0 THEN DAYS_CURRENT - DAYS_ABERTURA
```

**Protocolo Finalizado**:
```sql
WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_ABERTURA
```

**Exemplo Real**:
| NR_PROT | DATA_ABERTURA | DT_STATUS | FINISHED | SLA |
|---------|---------------|-----------|----------|-----|
| PR0309572023 | 26/06/2023 | 21/07/2023 | SIM | 25 |

Cálculo: DAYS(21/07/2023) - DAYS(26/06/2023) = 25 dias

## Exemplos Práticos no Resultado

Dados reais mostrando diferentes SLAs:

| NR_PROT | CATEGORIA_STATUS | SLA | SLA_OFICIAL | Análise |
|---------|------------------|-----|-------------|---------|
| PR0309572023 | Fora do Prazo | 25 | 15 | 25 > 15 (Atrasado) |
| PR0310732023 | Fora do Prazo | 27 | 15 | 27 > 15 (Atrasado) |
| PR0311262023 | Sem pagamento | 52 | 15 | Boleto vencido |
| PR0311962023 | Sem pagamento | 52 | 15 | Boleto vencido |

## Observações Técnicas

- Função DAYS() converte datas para números inteiros
- DAYS_CURRENT representa DAYS(CURRENT DATE)
- SLA = 0 indica casos especiais que não contam para métricas
- Protocolos em andamento têm SLA crescente diariamente
- Fundamental para análise de performance operacional

---
