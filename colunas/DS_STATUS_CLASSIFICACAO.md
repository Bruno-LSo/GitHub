DS_STATUS_CLASSIFICACAO.md

# DS_STATUS_CLASSIFICACAO - Classificação do Status

## Descrição Geral

A coluna `DS_STATUS_CLASSIFICACAO` categoriza todos os protocolos em três grandes grupos operacionais: Fechado, Exigência ou Aberto, facilitando análises de backlog e status operacional.

## Origem dos Dados

### Base de Classificação
- **Coluna de Origem**: `DS_STATUS` (já transformado)
- **Tipo de Dado**: VARCHAR - Classificação em três categorias

## Regra de Classificação

A query implementa uma lógica simples de três categorias:

```sql
-- No SELECT final:
CASE
    WHEN DS_STATUS IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida') 
        THEN 'Fechado'
    WHEN DS_STATUS IN ('Exigência Atendida', 'Documentação Pendente - Ver Exigências/Observações') 
        THEN 'Exigência'
    ELSE 'Aberto'
END AS DS_STATUS_Classificacao
```

## Categorias Definidas

### 1. FECHADO
Status que indicam conclusão definitiva do protocolo:
```sql
WHEN DS_STATUS IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida')
```

**Exemplos Reais**:
| NR_PROT | DS_STATUS | DS_STATUS_CLASSIFICACAO |
|---------|-----------|-------------------------|
| PR0309572023 | Solicitação Deferida | Fechado |
| PR0310732023 | Solicitação Deferida | Fechado |
| PR0311262023 | Solicitação Cancelada | Fechado |
| PR0311962023 | Solicitação Cancelada | Fechado |

### 2. EXIGÊNCIA
Status que indicam pendência documental:
```sql
WHEN DS_STATUS IN ('Exigência Atendida', 'Documentação Pendente - Ver Exigências/Observações')
```

**Características**:
- Protocolo aguarda ação do solicitante
- Documentação adicional necessária
- Processo temporariamente suspenso

### 3. ABERTO
Todos os demais status não finalizados:
```sql
ELSE 'Aberto'
```

**Status incluídos**:
- Aguardando Análise
- Aguardando Pagamento
- Solicitação Enviada
- Solicitação enviada à câmara especializada
- Análise Aprovada
- Qualquer outro status não listado anteriormente

## Utilização para Análise

### Métricas Operacionais
A classificação permite calcular:
- Taxa de conclusão: protocolos "Fechado" / total
- Volume em processamento: protocolos "Aberto"
- Pendências com cliente: protocolos "Exigência"

### Exemplo de Distribuição
Baseado nos dados de exemplo:
| DS_STATUS_CLASSIFICACAO | Quantidade | Percentual |
|------------------------|------------|------------|
| Fechado | 4 | 100% |
| Aberto | 0 | 0% |
| Exigência | 0 | 0% |

## Relacionamento com Outras Colunas

### DT_FINALIZACAO
Apenas protocolos classificados como "Fechado" possuem data de finalização:
```sql
WHEN DS_STATUS IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida')
THEN DT_STATUS_FMT
ELSE NULL
```

### FINALIZADO
Correlação direta com o campo FINISHED:
- "Fechado" geralmente tem FINALIZADO = "SIM"
- "Aberto" e "Exigência" têm FINALIZADO = "NÃO"

## Observações Técnicas

- A classificação é mutuamente exclusiva - cada protocolo pertence a apenas uma categoria
- A ordem de avaliação no CASE é importante
- Esta coluna é fundamental para:
  - Dashboards de acompanhamento operacional
  - Cálculo de backlog
  - Análise de eficiência do atendimento
- Simplifica filtros em ferramentas de BI

---

