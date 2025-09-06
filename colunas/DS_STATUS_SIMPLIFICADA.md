# DS_STATUS_SIMPLIFICADA - Status Simplificado

## Descrição Geral

A coluna `DS_STATUS_SIMPLIFICADA` agrupa e simplifica os diversos status possíveis do sistema em categorias padronizadas para facilitar análises gerenciais e relatórios consolidados.

## Origem dos Dados

### Base de Transformação
- **Coluna de Origem**: `DS_STATUS` (já transformado)
- **Tipo de Dado**: VARCHAR - Status simplificado para análise

## Regras de Transformação

A query aplica um mapeamento específico para cada status:

```sql
-- No SELECT final:
CASE
    WHEN DS_STATUS = 'Solicitação Deferida' THEN 'Solicitação Deferida'
    WHEN DS_STATUS = 'Solicitação Cancelada' THEN 'Solicitação Cancelada'
    WHEN DS_STATUS = 'Solicitação Indeferida' THEN 'Solicitação Indeferida'
    WHEN DS_STATUS = 'Aguardando Análise' THEN 'Aguardando Análise'
    WHEN DS_STATUS = 'Aguardando doctos. da IE' THEN 'Aguardando doctos. da IE'
    WHEN DS_STATUS = 'Aguardando Pagamento' THEN 'Aguardando Pagamento'
    WHEN DS_STATUS = 'Solicitação enviada à câmara especializada' THEN 'Solicitação enviada à câmara especializada'
    WHEN DS_STATUS = 'ExigÃªncia Atendida' THEN 'Aguardando Análise'
    WHEN DS_STATUS = 'Análise Aprovada' THEN 'Aguardando Análise'
    WHEN DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
    WHEN DS_STATUS = 'Solicitação Aguardando Retorno Banco' THEN 'Aguardando Pagamento'
    WHEN DS_STATUS = 'Solicitação Enviada' THEN 'Aguardando Pagamento'
    ELSE DS_STATUS
END AS DS_STATUS_Simplificada
```

## Mapeamentos de Simplificação

### Status que Mantêm o Nome Original:
- Solicitação Deferida → Solicitação Deferida
- Solicitação Cancelada → Solicitação Cancelada
- Solicitação Indeferida → Solicitação Indeferida
- Aguardando Análise → Aguardando Análise
- Aguardando Pagamento → Aguardando Pagamento

### Status Agrupados em "Aguardando Análise":
```sql
WHEN DS_STATUS = 'Exigência Atendida' THEN 'Aguardando Análise'
WHEN DS_STATUS = 'Análise Aprovada' THEN 'Aguardando Análise'
```

### Status Agrupados em "Exigência":
```sql
WHEN DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
```

### Status Agrupados em "Aguardando Pagamento":
```sql
WHEN DS_STATUS = 'Solicitação Aguardando Retorno Banco' THEN 'Aguardando Pagamento'
WHEN DS_STATUS = 'Solicitação Enviada' THEN 'Aguardando Pagamento'
```

## Exemplos Práticos no Resultado

Dados reais mostrando a simplificação:

| NR_PROT | DS_STATUS | DS_STATUS_SIMPLIFICADA |
|---------|-----------|------------------------|
| PR0309572023 | Solicitação Deferida | Solicitação Deferida |
| PR0310732023 | Solicitação Deferida | Solicitação Deferida |
| PR0311262023 | Solicitação Cancelada | Solicitação Cancelada |
| PR0311962023 | Solicitação Cancelada | Solicitação Cancelada |

## Relacionamento com Outras Colunas

### DS_STATUS_CLASSIFICACAO
O status simplificado influencia a classificação:
- Status finalizados (Deferida, Cancelada, Indeferida) → "Fechado"
- Status de exigência → "Exigência"
- Demais status → "Aberto"

### Análise Gerencial
A simplificação permite:
- Agrupar status similares para métricas consolidadas
- Reduzir a complexidade de relatórios
- Facilitar a compreensão por usuários não técnicos

## Observações Técnicas

- A simplificação preserva os status principais sem perda de informação crítica
- Status com ações pendentes similares são agrupados (ex: "Exigência Atendida" e "Análise Aprovada" viram "Aguardando Análise")
- O ELSE mantém o status original para casos não mapeados
- Esta coluna é essencial para dashboards executivos e análises de volume

---
