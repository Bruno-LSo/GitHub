# DS_STATUS - Descrição do Status

## Descrição Geral

A coluna `DS_STATUS` apresenta o status atual do protocolo com regras de negócio aplicadas que modificam o status original baseado em condições de pagamento e finalização.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_TIPO_STATUS_PROTOCOLO`
- **Coluna de Origem**: `DS_STATUS`
- **Tipo de Dado**: VARCHAR - Descrição textual do status

### Obtenção através de JOINs

```sql
-- Obtém o último status através da CTE ULTIMO_STATUS:
INNER JOIN ULTIMO_STATUS US ON US.CD_PROT = P.CD_PROT AND US.RN = 1
INNER JOIN DB2INET.TB_TIPO_STATUS_PROTOCOLO TSP ON TSP.CD_TP_STATUS_PROT = US.CD_TP_STATUS_PROT
```

## Regras de Transformação

A query aplica transformações complexas ao status original:

```sql
-- No SELECT final:
CASE
    -- Regra 1: Cancelamento por finalização antes do pagamento
    WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO
        THEN 'Solicitação Cancelada'
    
    -- Regra 2: Aguardando Pagamento
    WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NULL AND DS_STATUS = 'Solicitação Enviada'
        THEN 'Aguardando Pagamento'
    
    -- Regra 3: Aguardando Análise
    WHEN NR_BOLETO IS NULL AND FINISHED = 0 AND DS_STATUS = 'Solicitação Enviada'
        THEN 'Aguardando Análise'
    
    -- Regra 4: Mantém status original
    ELSE DS_STATUS
END AS DS_STATUS
```

## Exemplos Práticos de Transformação

### Regra 1 - Solicitação Cancelada
**Condição**: Protocolo finalizado antes do pagamento
```sql
WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO
```
**Exemplo**: Protocolo com pagamento em 29/06/2023 mas finalizado em 18/08/2023

### Regra 2 - Aguardando Pagamento
**Condição**: Boleto emitido sem pagamento
```sql
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NULL AND DS_STATUS = 'Solicitação Enviada'
```
**Exemplo Real**:
| NR_PROT | NR_BOLETO | DT_PAGAMENTO | DS_STATUS (Original) | DS_STATUS (Transformado) |
|---------|-----------|--------------|---------------------|-------------------------|
| PR0311262023 | 28027180231412766 | NULL | Solicitação Enviada | Aguardando Pagamento |

### Regra 3 - Aguardando Análise
**Condição**: Sem boleto e não finalizado
```sql
WHEN NR_BOLETO IS NULL AND FINISHED = 0 AND DS_STATUS = 'Solicitação Enviada'
```

## Valores Possíveis

Baseando-se na tabela `TB_TIPO_STATUS_PROTOCOLO` e nas transformações:

### Status Originais (exemplos):
- Solicitação Enviada
- Solicitação Deferida
- Solicitação Cancelada
- Solicitação Indeferida
- Documentação Pendente - Ver Exigências/Observações
- Exigência Atendida
- Análise Aprovada

### Status Transformados pela Query:
- Aguardando Pagamento (transformado de "Solicitação Enviada")
- Aguardando Análise (transformado de "Solicitação Enviada")
- Solicitação Cancelada (quando finalizado antes do pagamento)

## Exemplos no Resultado Final

Dados reais mostrando as transformações:

| NR_PROT | DS_STATUS | DS_OBSERVACOES | FINISHED | NR_BOLETO |
|---------|-----------|----------------|----------|-----------|
| PR0309572023 | Solicitação Deferida | Caso venha a realizar... | SIM | NULL |
| PR0310732023 | Solicitação Deferida | Efetivado o seu registro... | SIM | 28027180231410448 |
| PR0311262023 | Solicitação Cancelada | Boleto vencido: 28027180231412766... | SIM | 28027180231412766 |

## Relacionamento com Outras Colunas

### DS_STATUS_SIMPLIFICADA
O `DS_STATUS` é simplificado para análise gerencial, agrupando status similares.

### DS_STATUS_CLASSIFICACAO
Baseado no `DS_STATUS`, os protocolos são classificados em:
- "Fechado": Solicitação Deferida, Cancelada ou Indeferida
- "Exigência": Documentação Pendente ou Exigência Atendida
- "Aberto": Demais status

### DT_FINALIZACAO
Preenchida apenas quando:
```sql
WHEN DS_STATUS IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida')
THEN DT_STATUS_FMT
```

## Observações Técnicas

- A transformação do status ocorre em tempo de execução da query
- O status original é preservado na tabela base
- As regras refletem o fluxo operacional do CREA-SP
- A ordem das condições no CASE é importante - a primeira condição verdadeira determina o resultado
- Status "Solicitação Enviada" pode ser transformado em três diferentes status dependendo das condições