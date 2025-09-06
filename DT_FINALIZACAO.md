# DT_FINALIZACAO - Data de Finalização

## Descrição Geral

A coluna `DT_FINALIZACAO` apresenta a data em que o protocolo foi definitivamente concluído, seja por deferimento, indeferimento ou cancelamento.

## Origem dos Dados

### Cálculo Condicional
- **Base**: `DT_STATUS` quando o status é conclusivo
- **Tipo de Dado**: VARCHAR - Data formatada DD/MM/YYYY ou NULL

## Regra de Preenchimento

```sql
-- No SELECT final:
CASE
    WHEN DS_STATUS IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida')
    THEN DT_STATUS_FMT
    ELSE NULL
END AS DT_FINALIZACAO
```

## Status que Geram Data de Finalização

### Status Conclusivos
1. **Solicitação Deferida** - Aprovação do protocolo
2. **Solicitação Cancelada** - Cancelamento por qualquer motivo
3. **Solicitação Indeferida** - Negação do protocolo

### Status Não Conclusivos (DT_FINALIZACAO = NULL)
- Aguardando Análise
- Aguardando Pagamento
- Exigência Atendida
- Documentação Pendente
- Qualquer outro status em andamento

## Exemplos Práticos no Resultado

Dados reais mostrando finalizações:

| NR_PROT | DS_STATUS | DT_STATUS | DT_FINALIZACAO | FINALIZADO |
|---------|-----------|-----------|----------------|------------|
| PR0309572023 | Solicitação Deferida | 21/07/2023 | 21/07/2023 | SIM |
| PR0310732023 | Solicitação Deferida | 26/07/2023 | 26/07/2023 | SIM |
| PR0311262023 | Solicitação Cancelada | 18/08/2023 | 18/08/2023 | SIM |
| PR0311962023 | Solicitação Cancelada | 18/08/2023 | 18/08/2023 | SIM |

## Relação com Outras Colunas

### DS_STATUS_CLASSIFICACAO
Protocolos com DT_FINALIZACAO preenchida sempre têm:
```sql
DS_STATUS_CLASSIFICACAO = 'Fechado'
```

### FINALIZADO
Protocolos com DT_FINALIZACAO geralmente têm:
- FINALIZADO = 'SIM'
- FINISHED = 1

### DT_STATUS
DT_FINALIZACAO é sempre igual a DT_STATUS quando preenchida:
- Copia o valor de DT_STATUS_FMT
- Representa o momento da última alteração significativa

## Casos Especiais

### Cancelamento por Boleto Vencido
```
DT_FINALIZACAO = Data do cancelamento (não da vencimento)
```

**Exemplo**:
| NR_PROT | DS_OBSERVACOES | DT_FINALIZACAO |
|---------|----------------|----------------|
| PR0311262023 | Boleto vencido: ... - Vencto.: 04/07/2023 | 18/08/2023 |

Note que a data de finalização (18/08/2023) é diferente da data de vencimento (04/07/2023).

## Utilização para Análise

### Métricas Possíveis
- Taxa de conclusão por período
- Tempo médio até finalização
- Volume de finalizações por tipo (deferido/indeferido/cancelado)
- Análise de sazonalidade

## Observações Técnicas

- Campo opcional (NULL para protocolos em andamento)
- Sempre no formato DD/MM/YYYY quando preenchido
- Imutável após preenchimento
- Fundamental para relatórios de fechamento mensal
- Base para cálculo de indicadores de eficiência

---
