# DS_STATUS_SIMPLIFICADA - Status Simplificado

## Descrição Geral

A coluna `DS_STATUS_SIMPLIFICADA` agrupa e simplifica os diversos status possíveis do sistema em categorias padronizadas para facilitar análises gerenciais e relatórios consolidados.

## Origem dos Dados

### Base de Transformação
- **Coluna de Origem**: `DS_STATUS` (já transformado pelas regras de negócio)
- **Tipo de Dado**: VARCHAR - Status simplificado para análise
- **Tabela Base Original**: `DB2INET.TB_TIPO_STATUS_PROTOCOLO` através do campo `DS_STATUS`

## Regras Completas de Transformação

A query aplica um mapeamento específico e exaustivo para cada status:

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
    WHEN DS_STATUS = 'Exigência Atendida' THEN 'Aguardando Análise'
    WHEN DS_STATUS = 'Análise Aprovada' THEN 'Aguardando Análise'
    WHEN DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
    WHEN DS_STATUS = 'Solicitação Aguardando Retorno Banco' THEN 'Aguardando Pagamento'
    WHEN DS_STATUS = 'Solicitação Enviada' THEN 'Aguardando Pagamento'
    ELSE DS_STATUS
END AS DS_STATUS_Simplificada
```

## Mapeamentos Completos de Simplificação

### 1. Status que Mantêm o Nome Original (Sem Transformação):

```sql
WHEN DS_STATUS = 'Solicitação Deferida' THEN 'Solicitação Deferida'
WHEN DS_STATUS = 'Solicitação Cancelada' THEN 'Solicitação Cancelada'
WHEN DS_STATUS = 'Solicitação Indeferida' THEN 'Solicitação Indeferida'
WHEN DS_STATUS = 'Aguardando Análise' THEN 'Aguardando Análise'
WHEN DS_STATUS = 'Aguardando Pagamento' THEN 'Aguardando Pagamento'
WHEN DS_STATUS = 'Aguardando doctos. da IE' THEN 'Aguardando doctos. da IE'
WHEN DS_STATUS = 'Solicitação enviada à câmara especializada' THEN 'Solicitação enviada à câmara especializada'
```

**Exemplos Reais**:
| NR_PROT | DS_STATUS | DS_STATUS_SIMPLIFICADA |
|---------|-----------|------------------------|
| PR0309572023 | Solicitação Deferida | Solicitação Deferida |
| PR0310732023 | Solicitação Deferida | Solicitação Deferida |
| PR0311262023 | Solicitação Cancelada | Solicitação Cancelada |
| PR0311962023 | Solicitação Cancelada | Solicitação Cancelada |

### 2. Status Agrupados em "Aguardando Análise":

```sql
WHEN DS_STATUS = 'Exigência Atendida' THEN 'Aguardando Análise'
WHEN DS_STATUS = 'Análise Aprovada' THEN 'Aguardando Análise'
```

**Lógica**: Estes status indicam que o protocolo passou por uma etapa mas ainda aguarda análise final, sendo consolidados em uma única categoria.

### 3. Status Transformado em "Exigência":

```sql
WHEN DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
```

**Lógica**: Simplifica o nome longo do status para facilitar visualização em relatórios.

### 4. Status Agrupados em "Aguardando Pagamento":

```sql
WHEN DS_STATUS = 'Solicitação Aguardando Retorno Banco' THEN 'Aguardando Pagamento'
WHEN DS_STATUS = 'Solicitação Enviada' THEN 'Aguardando Pagamento'
```

**Importante**: Note que "Solicitação Enviada" é transformada em "Aguardando Pagamento" nesta coluna, diferente da transformação em DS_STATUS que pode resultar em "Aguardando Análise" dependendo de outras condições.

### 5. Cláusula ELSE - Mantém Status Original:

```sql
ELSE DS_STATUS
```

**Lógica**: Qualquer status não explicitamente mapeado nas regras anteriores é mantido sem alteração. Isso garante que novos status ou status não previstos ainda sejam exibidos.

## Interação com DS_STATUS

É importante notar que DS_STATUS_SIMPLIFICADA trabalha sobre o DS_STATUS já transformado, que por sua vez pode ter sido modificado por estas regras:

```sql
-- Transformações em DS_STATUS que afetam DS_STATUS_SIMPLIFICADA:
CASE
    WHEN DT_PAGAMENTO IS NOT NULL AND FINISHED = 1 AND DAYS_STATUS < DAYS_PAGAMENTO
        THEN 'Solicitação Cancelada'
    WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NULL AND DS_STATUS = 'Solicitação Enviada'
        THEN 'Aguardando Pagamento'
    WHEN NR_BOLETO IS NULL AND FINISHED = 0 AND DS_STATUS = 'Solicitação Enviada'
        THEN 'Aguardando Análise'
    ELSE DS_STATUS
END AS DS_STATUS
```

## Exemplos de Transformação em Cascata

### Exemplo 1: Solicitação Enviada → Aguardando Pagamento → Aguardando Pagamento

**Cenário**: Protocolo com boleto sem pagamento
- DS_STATUS original (TB_TIPO_STATUS_PROTOCOLO): "Solicitação Enviada"
- DS_STATUS transformado: "Aguardando Pagamento" (por ter NR_BOLETO NOT NULL e DT_PAGAMENTO IS NULL)
- DS_STATUS_SIMPLIFICADA: "Aguardando Pagamento" (mantém)

### Exemplo 2: Solicitação Enviada → Aguardando Análise → Aguardando Análise

**Cenário**: Protocolo sem boleto e não finalizado
- DS_STATUS original: "Solicitação Enviada"
- DS_STATUS transformado: "Aguardando Análise" (por ter NR_BOLETO IS NULL e FINISHED = 0)
- DS_STATUS_SIMPLIFICADA: "Aguardando Análise" (mantém)

## Lista Completa de Status Possíveis em DS_STATUS_SIMPLIFICADA

Baseado nas regras da query, os valores finais possíveis são:

1. **Solicitação Deferida**
2. **Solicitação Cancelada**
3. **Solicitação Indeferida**
4. **Aguardando Análise**
5. **Aguardando doctos. da IE**
6. **Aguardando Pagamento**
7. **Solicitação enviada à câmara especializada**
8. **Exigência**
9. **Qualquer outro status** (mantido pelo ELSE)

## Relacionamento com Outras Colunas

### DS_STATUS_CLASSIFICACAO

O status simplificado influencia diretamente a classificação:

```sql
CASE
    WHEN DS_STATUS IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida') 
        THEN 'Fechado'
    WHEN DS_STATUS IN ('Exigência Atendida', 'Documentação Pendente - Ver Exigências/Observações') 
        THEN 'Exigência'
    ELSE 'Aberto'
END AS DS_STATUS_Classificacao
```

Note que DS_STATUS_CLASSIFICACAO usa DS_STATUS (não simplificado) para sua lógica.

## Análise Gerencial

A simplificação permite:
- **Redução de complexidade**: De potencialmente dezenas de status para aproximadamente 8 categorias principais
- **Agrupamento lógico**: Status com significado operacional similar são consolidados
- **Facilita dashboards**: Menos categorias para visualização
- **Mantém flexibilidade**: Cláusula ELSE preserva status não mapeados

## Observações Técnicas

- A simplificação ocorre após todas as transformações de DS_STATUS
- A ordem das condições no CASE é irrelevante pois são mutuamente exclusivas
- Status com acentuação devem corresponder exatamente (case-sensitive para acentos)
- O ELSE garante que a query não falhe com status novos ou inesperados
- Esta coluna é essencial para relatórios executivos onde detalhamento excessivo prejudica a análise

---