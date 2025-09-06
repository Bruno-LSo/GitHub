# NM_SUBTP_PROT - Nome do Subtipo de Protocolo

## Descrição Geral

A coluna `NM_SUBTP_PROT` detalha o subtipo específico da solicitação dentro de cada tipo de protocolo, fornecendo granularidade para análise e determinação de prazos de atendimento.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_SUBTIPO_PROTOCOLO`
- **Coluna de Origem**: `NM_SUBTP_PROT`
- **Tipo de Dado**: VARCHAR - Descrição do subtipo de protocolo

### Obtenção através de JOIN

```sql
-- Na query, obtido através do JOIN:
FROM DB2INET.TB_PROTOCOLO_CREANET P
INNER JOIN DB2INET.TB_SUBTIPO_PROTOCOLO STP ON P.CD_SUBTP_PROT = STP.CD_SUBTP_PROT
```

## Exemplos de Subtipos

Baseando-se nos dados reais da tabela:

| CD_SUBTP_PROT | NM_SUBTP_PROT | CD_TP_PROT |
|---------------|---------------|------------|
| 1 | Registro com atestado - RU | 1 |
| 2 | Registro com diploma - RU | 1 |
| 25 | CAT Com Registro de Atestado - Atividade Concluída | 3 |

## Utilização Principal - SLA_OFICIAL

O subtipo é o principal determinante do prazo oficial de atendimento:

```sql
-- Cálculo do SLA_OFICIAL na CTE PROTOCOLO_COMPLETO:
CASE
    WHEN STP.NM_SUBTP_PROT = 'Alteração de Registro de Empresa' THEN 20
    WHEN STP.NM_SUBTP_PROT IN ('Baixa Art - BAIXA DO RESP. PERANTE EMPR. CONTRATADA',
                                'Baixa Art - PARALISAÇÃO DA OBRA/SERVIÇO',
                                'Baixa Art - RESCISÃO CONTRATUAL',
                                'Baixa Art - SUBSTITUIÇÃO DO RESPONSÁVEL TÉCNICO') THEN 10
    WHEN STP.NM_SUBTP_PROT = 'CADASTRO DE IES' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Interrupção de Registro' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Reativar Registro' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Registro com diploma CREANET' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Registro de Empresa' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Segunda Via' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Visto de Profissional' THEN 15
    ELSE 20  -- Padrão para subtipos não mapeados
END AS SLA_OFICIAL
```

## Exemplos Práticos no Resultado

Dados reais da query com seus respectivos SLAs:

| NR_PROT | NM_SUBTP_PROT | SLA_OFICIAL | CATEGORIA_STATUS |
|---------|---------------|-------------|------------------|
| PR0309572023 | Interrupção de Registro | 15 | Fora do Prazo |
| PR0310732023 | Reativar Registro | 15 | Fora do Prazo |
| PR0311262023 | Reativar Registro | 15 | Sem pagamento |
| PR0311962023 | Reativar Registro | 15 | Sem pagamento |

## Categorização por Prazo

### Subtipos com 10 dias de prazo:
- Baixa Art - BAIXA DO RESP. PERANTE EMPR. CONTRATADA
- Baixa Art - PARALISAÇÃO DA OBRA/SERVIÇO
- Baixa Art - RESCISÃO CONTRATUAL
- Cancelar Art - CONTRATO NÃO FOI EXECUTADO

### Subtipos com 15 dias de prazo:
- CADASTRO DE IES
- Interrupção de Registro
- Reativar Registro
- Registro com diploma CREANET
- Visto de Profissional

### Subtipos com 20 dias de prazo:
- Alteração de Registro de Empresa
- Cancelamento de Registro de Empresa
- CAT Com Registro de Atestado (todos os tipos)
- Registro de Empresa
- Segunda Via
- Validação Prévia para Cartório

## Relacionamento com Outras Colunas

### SLA e CATEGORIA_STATUS
O `NM_SUBTP_PROT` através do `SLA_OFICIAL` determina se um protocolo está:
- "Dentro do Prazo": quando SLA <= SLA_OFICIAL
- "Fora do Prazo": quando SLA > SLA_OFICIAL

### Regra Especial para Registro de Empresa

```sql
-- No cálculo do SLA_PROTOCOLO:
WHEN NM_TP_PROT = 'Empresa' 
     AND NM_SUBTP_PROT = 'Registro de Empresa' 
     AND DS_STATUS = 'Solicitação Deferida' 
THEN 0
```

## Observações Técnicas

- Cada protocolo possui obrigatoriamente um subtipo
- O subtipo deve ser compatível com o tipo de protocolo (relação CD_TP_PROT)
- O campo `NM_SUBTP_PROT` é essencial para determinar os prazos regulamentares
- Subtipos não mapeados no CASE recebem o prazo padrão de 20 dias

---
