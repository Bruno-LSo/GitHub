# NR_BOLETO - Número do Boleto

## Descrição Geral

A coluna `NR_BOLETO` apresenta o número do boleto bancário associado ao protocolo, quando aplicável, indicando que há cobrança relacionada ao serviço solicitado.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_BOLETO`
- **Coluna de Origem**: `NR_BOLETO`
- **Tipo de Dado**: VARCHAR - Número identificador do boleto

### Obtenção através de JOIN

```sql
-- Na CTE PROTOCOLO_COMPLETO:
LEFT JOIN DB2INET.TB_BOLETO BOLETO ON BOLETO.CD_PROC = P.NR_PROT

-- Formatação:
TO_CHAR(BOLETO.NR_BOLETO) AS NR_BOLETO_FMT

-- No SELECT final:
NR_BOLETO_FMT AS NR_BOLETO
```

## Estrutura do Número

Baseando-se nos dados reais:

| NR_BOLETO | Estrutura |
|-----------|-----------|
| 28027180231757298 | Código de 17 dígitos |
| 28027180231410448 | Código de 17 dígitos |
| 28027180231412766 | Código de 17 dígitos |
| 28027180231414609 | Código de 17 dígitos |

Padrão identificado: 280271802...

## Utilização nas Regras de Negócio

### 1. Identificação de Aguardando Pagamento

```sql
-- Na coluna DS_STATUS:
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NULL AND DS_STATUS = 'Solicitação Enviada'
    THEN 'Aguardando Pagamento'
```

### 2. Identificação de Aguardando Análise

```sql
-- Na coluna DS_STATUS:
WHEN NR_BOLETO IS NULL AND FINISHED = 0 AND DS_STATUS = 'Solicitação Enviada'
    THEN 'Aguardando Análise'
```

### 3. Verificação de Protocolos Sem Pagamento

```sql
-- Em CATEGORIA_STATUS e GRE:
WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
     AND DT_PAGAMENTO IS NULL 
     AND NR_BOLETO IS NOT NULL
THEN 'Sem pagamento'
```

### 4. Identificação de Sem Distribuição

```sql
-- Na coluna GRE:
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NOT NULL
    AND COALESCE(DEPARTMENT, '') = ''
    AND COALESCE(GRE_DESC, '') = ''
    AND COALESCE(CIDADE, '') = ''
    AND COALESCE(ATRIBUIDO, '') = ''
THEN 'Sem distribuição'
```

## Exemplos Práticos no Resultado

Dados reais mostrando boletos:

| NR_PROT | NR_BOLETO | DT_PAGAMENTO | DS_STATUS | CATEGORIA_STATUS |
|---------|-----------|--------------|-----------|------------------|
| PR0309572023 | NULL | NULL | Solicitação Deferida | Fora do Prazo |
| PR0310732023 | 28027180231410448 | 29/06/2023 | Solicitação Deferida | Fora do Prazo |
| PR0311262023 | 28027180231412766 | NULL | Solicitação Cancelada | Sem pagamento |
| PR0311962023 | 28027180231414609 | NULL | Solicitação Cancelada | Sem pagamento |

## Relação com Status de Pagamento

### Protocolos com Boleto e Pagamento
- Processamento normal concluído
- SLA calculado a partir da data de pagamento

### Protocolos com Boleto sem Pagamento
- Status: "Aguardando Pagamento" ou "Sem pagamento"
- Podem ser cancelados por vencimento

### Protocolos sem Boleto
- Podem não requerer pagamento
- Status: "Aguardando Análise" quando não finalizados

## Observações em Cancelamentos

Boletos vencidos aparecem nas observações:
```
"Boleto vencido: 28027180231412766 - Vencto.: 04/07/2023 00:00:00"
```

## Observações Técnicas

- Campo opcional (pode ser NULL)
- Relacionamento através de CD_PROC = NR_PROT
- Convertido para string com TO_CHAR()
- Fundamental para controle financeiro
- Indica necessidade de pagamento para processamento
- Base para relatórios de inadimplência