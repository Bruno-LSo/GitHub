# SLA - Tempo de Atendimento

## Descrição Geral

A coluna `SLA` calcula o tempo efetivo de atendimento do protocolo em dias, considerando diferentes cenários de pagamento, finalização, cancelamento e tipos específicos de protocolo.

## Origem dos Dados

### Cálculo Dinâmico Baseado em Múltiplas Tabelas
- **Tipo de Dado**: INTEGER - Número de dias
- **Tabelas de Origem**:
  - `DB2INET.TB_PROTOCOLO_CREANET`: DT_ABERTURA, FINISHED, CD_TP_PROT, CD_SUBTP_PROT
  - `DB2INET.TB_STATUS_PROTOCOLO_CREANET`: DT_STATUS, DS_OBSERVACOES
  - `DB2INET.TB_BOLETO`: DT_PAGAMENTO
  - `DB2INET.TB_TIPO_PROTOCOLO`: NM_TP_PROT
  - `DB2INET.TB_SUBTIPO_PROTOCOLO`: NM_SUBTP_PROT
  - `DB2INET.TB_TIPO_STATUS_PROTOCOLO`: DS_STATUS

## Regras Completas de Cálculo

A query implementa um CASE complexo com todas estas regras em ordem de prioridade:

```sql
-- No SELECT final, coluna SLA:
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

## Detalhamento de Cada Regra

### Regra 1: SLA = 0 para Cancelamento por Boleto Vencido

```sql
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 0
```

**Condições**:
- Status = "Solicitação Cancelada"
- Protocolo finalizado (FINISHED = 1)
- Observações contêm "Boleto Vencido"

**Exemplo Real**:
| NR_PROT | DS_STATUS | FINISHED | DS_OBSERVACOES | SLA |
|---------|-----------|----------|----------------|-----|
| PR0311262023 | Solicitação Cancelada | SIM | Boleto vencido: 28027180231412766 - Vencto.: 04/07/2023 00:00:00 | 0 |
| PR0311962023 | Solicitação Cancelada | SIM | Boleto vencido: 28027180231414609 - Vencto.: 04/07/2023 00:00:00 | 0 |

**Lógica**: Protocolos cancelados por não pagamento não contam para métricas de SLA.

### Regra 2: SLA = 0 para Finalização Antes do Pagamento

```sql
WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO 
THEN 0
```

**Condições**:
- Existe pagamento (DT_PAGAMENTO IS NOT NULL)
- Protocolo finalizado (FINISHED = 1)
- Data do último status anterior à data de pagamento

**Lógica**: Se o protocolo foi finalizado antes mesmo do pagamento ser processado, indica situação anormal que não deve contar no SLA.

### Regra 3: SLA = 0 para Registro de Empresa Deferido

```sql
WHEN NM_TP_PROT = 'Empresa' AND NM_SUBTP_PROT = 'Registro de Empresa' 
     AND DS_STATUS = 'Solicitação Deferida' 
THEN 0
```

**Condições**:
- Tipo de protocolo = "Empresa"
- Subtipo = "Registro de Empresa"
- Status = "Solicitação Deferida"

**Lógica**: Regra de negócio específica onde registros de empresa deferidos não contam SLA (possivelmente por serem processados automaticamente ou terem fluxo diferenciado).

### Regra 4: Cálculo com Pagamento

```sql
WHEN DAYS_PAGAMENTO IS NOT NULL THEN
    CASE
        WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_PAGAMENTO
        ELSE DAYS_CURRENT - DAYS_PAGAMENTO
    END
```

**Subregra 4.1 - Protocolo Finalizado com Pagamento**:
```sql
WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_PAGAMENTO
```

**Exemplo Real**:
| NR_PROT | DT_PAGAMENTO | DT_STATUS | FINISHED | Cálculo | SLA |
|---------|--------------|-----------|----------|---------|-----|
| PR0310732023 | 29/06/2023 | 26/07/2023 | SIM | DAYS(26/07/2023) - DAYS(29/06/2023) | 27 |

**Subregra 4.2 - Protocolo em Andamento com Pagamento**:
```sql
ELSE DAYS_CURRENT - DAYS_PAGAMENTO
```

**Lógica**: Para protocolos pagos ainda em processamento, conta-se desde o pagamento até hoje.

### Regra 5: Cálculo sem Pagamento

```sql
ELSE
    CASE
        WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_ABERTURA
        ELSE DAYS_CURRENT - DAYS_ABERTURA
    END
```

**Subregra 5.1 - Protocolo Finalizado sem Pagamento**:
```sql
WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_ABERTURA
```

**Exemplo Real**:
| NR_PROT | DATA_ABERTURA | DT_STATUS | FINISHED | DT_PAGAMENTO | Cálculo | SLA |
|---------|---------------|-----------|----------|--------------|---------|-----|
| PR0309572023 | 26/06/2023 | 21/07/2023 | SIM | NULL | DAYS(21/07/2023) - DAYS(26/06/2023) | 25 |

**Subregra 5.2 - Protocolo em Andamento sem Pagamento**:
```sql
ELSE DAYS_CURRENT - DAYS_ABERTURA
```

**Lógica**: Para protocolos sem pagamento ainda em processamento, conta-se desde a abertura até a data atual.

## Conversão de Datas para Cálculo

A query utiliza a função DAYS() do DB2 para converter datas em números inteiros:

```sql
-- Na CTE PROTOCOLO_COMPLETO:
DAYS(P.DT_ABERTURA) AS DAYS_ABERTURA,
DAYS(US.DT_STATUS) AS DAYS_STATUS,
DAYS(CURRENT DATE) AS DAYS_CURRENT,
DAYS(BOLETO.DT_PAGAMENTO) AS DAYS_PAGAMENTO,
```

Esta conversão permite operações aritméticas diretas entre datas.

## Exemplos Práticos do Resultado Completo

Dados reais mostrando diferentes cenários de SLA:

| NR_PROT | CATEGORIA_STATUS | SLA | SLA_OFICIAL | NR_BOLETO | DT_PAGAMENTO | FINISHED | Análise |
|---------|------------------|-----|-------------|-----------|--------------|----------|---------|
| PR0309572023 | Fora do Prazo | 25 | 15 | NULL | NULL | SIM | Sem pagamento, finalizado: 25 dias |
| PR0310732023 | Fora do Prazo | 27 | 15 | 28027180231410448 | 29/06/2023 | SIM | Com pagamento, finalizado: 27 dias |
| PR0311262023 | Cancelado por não pagamento | 0 | 15 | 28027180231412766 | NULL | SIM | Boleto vencido: SLA zerado |
| PR0311962023 | Cancelado por não pagamento | 0 | 15 | 28027180231414609 | NULL | SIM | Boleto vencido: SLA zerado |

Note que os protocolos PR0311262023 e PR0311962023 têm SLA = 0 mas CATEGORIA_STATUS mostra "Cancelado por não pagamento", não "Sem pagamento", devido à regra específica de boleto vencido.

## Fluxo de Decisão do Cálculo

```
1. É cancelado com boleto vencido? → SLA = 0
2. Foi finalizado antes do pagamento? → SLA = 0  
3. É Empresa/Registro deferido? → SLA = 0
4. Tem pagamento?
   4.1. Finalizado? → DAYS_STATUS - DAYS_PAGAMENTO
   4.2. Em andamento? → DAYS_CURRENT - DAYS_PAGAMENTO
5. Sem pagamento?
   5.1. Finalizado? → DAYS_STATUS - DAYS_ABERTURA
   5.2. Em andamento? → DAYS_CURRENT - DAYS_ABERTURA
```

## Impacto do SLA em Outras Colunas

### CATEGORIA_STATUS

O SLA calculado é comparado com SLA_OFICIAL para determinar:

```sql
WHEN (SLA calculado) <= SLA_OFICIAL THEN 'Dentro do Prazo'
ELSE 'Fora do Prazo'
```

## Observações Técnicas

- A função DAYS() retorna um inteiro representando dias desde uma data de referência interna do DB2
- DAYS_CURRENT é calculado uma única vez na CTE para consistência
- SLA = 0 indica casos especiais que devem ser excluídos de métricas de performance
- Protocolos em andamento têm SLA crescente diariamente até finalização
- A ordem das condições no CASE é crucial - a primeira condição verdadeira determina o resultado
- O cálculo considera dias corridos, não dias úteis

---