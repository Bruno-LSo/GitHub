# CATEGORIA_STATUS - Categorização por Prazo e Pagamento

## Descrição Geral

A coluna `CATEGORIA_STATUS` classifica os protocolos em categorias operacionais baseadas em prazo, pagamento e situação, facilitando a identificação de protocolos críticos.

## Origem dos Dados

### Cálculo Baseado em Múltiplas Condições
- **Tipo de Dado**: VARCHAR - Categoria textual
- **Base**: Combinação de SLA, SLA_OFICIAL, pagamento e status

## Regras de Categorização

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
    ELSE 'Fora do Prazo'
END AS CATEGORIA_STATUS
```

## Categorias Definidas

### 1. Cancelado por não pagamento

**Condições**:
- Status = "Solicitação Cancelada"
- Protocolo finalizado (FINISHED = 1)
- Observações contêm "Boleto Vencido"

**Exemplo Real**:
| NR_PROT | DS_OBSERVACOES | CATEGORIA_STATUS |
|---------|----------------|------------------|
| PR0311262023 | Boleto vencido: 28027180231412766... | Cancelado por não pagamento |

### 2. Sem pagamento

**Condições**:
- Data atual > Data estimada de conclusão
- Sem pagamento registrado (DT_PAGAMENTO IS NULL)
- Boleto emitido (NR_BOLETO IS NOT NULL)

**Exemplo Real**:
| NR_PROT | DATA_EST_CONCLUSAO | NR_BOLETO | DT_PAGAMENTO | CATEGORIA_STATUS |
|---------|-------------------|-----------|--------------|------------------|
| PR0311262023 | 04/07/2023 | 28027180231412766 | NULL | Sem pagamento |

### 3. Dentro do Prazo

**Condição**: Tempo decorrido ≤ SLA_OFICIAL

```sql
WHEN (tempo_calculado) <= SLA_OFICIAL
THEN 'Dentro do Prazo'
```

### 4. Fora do Prazo

**Condição**: Tempo decorrido > SLA_OFICIAL

**Exemplos Reais**:
| NR_PROT | SLA | SLA_OFICIAL | CATEGORIA_STATUS |
|---------|-----|-------------|------------------|
| PR0309572023 | 25 | 15 | Fora do Prazo |
| PR0310732023 | 27 | 15 | Fora do Prazo |

## Cálculo do Tempo para Comparação

O tempo usado na comparação com SLA_OFICIAL segue a mesma lógica do SLA:

```sql
CASE
    WHEN DAYS_PAGAMENTO IS NOT NULL AND FINISHED = 1 
        THEN DAYS_STATUS - DAYS_PAGAMENTO
    WHEN DAYS_PAGAMENTO IS NOT NULL 
        THEN DAYS_CURRENT - DAYS_PAGAMENTO
    WHEN FINISHED = 1 
        THEN DAYS_STATUS - DAYS_ABERTURA
    ELSE DAYS_CURRENT - DAYS_ABERTURA
END
```

## Prioridade das Regras

A ordem de avaliação é crucial:
1. Primeiro verifica cancelamento por não pagamento
2. Depois verifica protocolos sem pagamento após prazo
3. Por último, compara com SLA_OFICIAL

## Observações Técnicas

- Categoria mutuamente exclusiva (apenas uma por protocolo)
- "Cancelado por não pagamento" tem prioridade máxima
- Essencial para dashboards de monitoramento
- Permite identificação rápida de protocolos críticos
- Base para alertas e ações corretivas

---
