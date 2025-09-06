# FINALIZADO - Indicador de Finalização

## Descrição Geral

A coluna `FINALIZADO` indica se o protocolo teve seu processamento concluído, apresentando "SIM" ou "NÃO" baseado no campo FINISHED da tabela de protocolos.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_PROTOCOLO_CREANET`
- **Coluna de Origem**: `FINISHED`
- **Tipo de Dado**: INTEGER (0 ou 1) transformado para VARCHAR

### Transformação

```sql
-- No SELECT final:
CASE WHEN FINISHED = 1 THEN 'SIM' ELSE 'NÃO' END AS FINALIZADO
```

## Valores Possíveis

| FINISHED (Original) | FINALIZADO (Transformado) |
|-------------------|--------------------------|
| 1 | SIM |
| 0 | NÃO |

## Utilização nas Regras de Negócio

### 1. Determinação de Status Cancelado

```sql
-- Na coluna DS_STATUS:
WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO
    THEN 'Solicitação Cancelada'
```

### 2. Identificação de Aguardando Análise

```sql
-- Na coluna DS_STATUS:
WHEN NR_BOLETO IS NULL AND FINISHED = 0 AND DS_STATUS = 'Solicitação Enviada'
    THEN 'Aguardando Análise'
```

### 3. Cálculo do SLA

O campo FINISHED determina como calcular o tempo de atendimento:

```sql
-- Para protocolos com pagamento:
WHEN DAYS_PAGAMENTO IS NOT NULL THEN
    CASE
        WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_PAGAMENTO  -- Usa data do status final
        ELSE DAYS_CURRENT - DAYS_PAGAMENTO  -- Usa data atual
    END
-- Para protocolos sem pagamento:
ELSE
    CASE
        WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_ABERTURA  -- Usa data do status final
        ELSE DAYS_CURRENT - DAYS_ABERTURA  -- Usa data atual
    END
```

### 4. Identificação de Cancelamento por Não Pagamento

```sql
-- Em CATEGORIA_STATUS, GRE e GRE_Simplificada:
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 'Cancelado por não pagamento'
```

## Exemplos Práticos no Resultado

Dados reais mostrando o indicador:

| NR_PROT | DS_STATUS | FINALIZADO | SLA | DT_FINALIZACAO |
|---------|-----------|------------|-----|----------------|
| PR0309572023 | Solicitação Deferida | SIM | 25 | 21/07/2023 |
| PR0310732023 | Solicitação Deferida | SIM | 27 | 26/07/2023 |
| PR0311262023 | Solicitação Cancelada | SIM | 0 | 18/08/2023 |
| PR0311962023 | Solicitação Cancelada | SIM | 0 | 18/08/2023 |

## Relação com Status

### Protocolos Finalizados (FINISHED = 1)
Geralmente associados aos status:
- Solicitação Deferida
- Solicitação Cancelada
- Solicitação Indeferida

### Protocolos Em Andamento (FINISHED = 0)
Associados aos status:
- Aguardando Análise
- Aguardando Pagamento
- Exigência Atendida
- Documentação Pendente

## Impacto em Outras Colunas

### DT_FINALIZACAO
Só é preenchida quando FINALIZADO = "SIM" e status é conclusivo

### SLA
- FINALIZADO = "SIM": usa data do último status para cálculo
- FINALIZADO = "NÃO": usa data atual para cálculo contínuo

### CATEGORIA_STATUS
Combinado com outras condições para determinar:
- "Cancelado por não pagamento" (requer FINISHED = 1)
- Status de processamento em andamento (FINISHED = 0)

## Filtros da Query

A query principal filtra apenas protocolos válidos:

```sql
WHERE P.DRAFT = 0  -- Exclui rascunhos
```

Não há filtro específico para FINISHED, incluindo tanto protocolos finalizados quanto em andamento.

## Observações Técnicas

- Campo binário (0 ou 1) transformado em texto legível
- Fundamental para cálculos de tempo e SLA
- Indica o estado operacional do protocolo
- Protocolos finalizados não sofrem mais alterações
- Usado em conjunto com DS_STATUS para validações de consistência

---