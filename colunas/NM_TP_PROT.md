# NM_TP_PROT - Nome do Tipo de Protocolo

## Descrição Geral

A coluna `NM_TP_PROT` apresenta a descrição textual do tipo de protocolo, classificando a natureza principal da solicitação realizada no sistema CREA-SP.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_TIPO_PROTOCOLO`
- **Coluna de Origem**: `NM_TP_PROT`
- **Tipo de Dado**: VARCHAR - Descrição do tipo de protocolo

### Obtenção através de JOIN

```sql
-- Na query, o NM_TP_PROT é obtido através do JOIN:
FROM DB2INET.TB_PROTOCOLO_CREANET P
INNER JOIN DB2INET.TB_TIPO_PROTOCOLO TP ON P.CD_TP_PROT = TP.CD_TP_PROT
```

## Valores Disponíveis

Baseando-se nos dados reais da tabela `TB_TIPO_PROTOCOLO`:

| CD_TP_PROT | NM_TP_PROT |
|------------|------------|
| 1 | Registro de Profissional |
| 2 | Baixa Art |
| 3 | Acervo Técnico |
| 4 | Cancelamento Art |
| 5 | Certidões |

## Exemplos Práticos no Resultado

Dados reais extraídos da query:

| NR_PROT | NM_TP_PROT | NM_SUBTP_PROT |
|---------|------------|---------------|
| PR0309572023 | Registro de Profissional | Interrupção de Registro |
| PR0310732023 | Registro de Profissional | Reativar Registro |
| PR0311262023 | Registro de Profissional | Reativar Registro |
| PR0311962023 | Registro de Profissional | Reativar Registro |

## Utilização nas Regras de Negócio

### 1. Identificação de Acervo Técnico

A query identifica funcionários que trabalham com Acervo Técnico:

```sql
-- Na CTE FUNCIONARIOS_ACERVO:
WHERE P.CD_TP_PROT IN (
    SELECT CD_TP_PROT 
    FROM DB2INET.TB_TIPO_PROTOCOLO
    WHERE NM_TP_PROT = 'Acervo Técnico'
)
```

### 2. Regra Especial para Registro de Empresa

No cálculo do SLA_PROTOCOLO, existe uma regra específica:

```sql
-- Quando é Empresa com Registro deferido, SLA = 0:
WHEN NM_TP_PROT = 'Empresa' 
     AND NM_SUBTP_PROT = 'Registro de Empresa' 
     AND DS_STATUS = 'Solicitação Deferida' 
THEN 0
```

## Relacionamento com Outras Colunas

### NM_SUBTP_PROT
O tipo de protocolo determina quais subtipos estão disponíveis. Por exemplo:
- Tipo: "Registro de Profissional" possui subtipos como "Reativar Registro", "Interrupção de Registro"
- Tipo: "Acervo Técnico" possui subtipos específicos para CAT

### SLA_OFICIAL
Embora o SLA_OFICIAL seja principalmente determinado pelo subtipo, o tipo de protocolo influencia indiretamente os prazos aplicáveis.

## Observações Técnicas

- O relacionamento é feito através do código `CD_TP_PROT` entre as tabelas
- Cada protocolo possui obrigatoriamente um tipo associado
- A descrição é padronizada e controlada pela tabela de domínio `TB_TIPO_PROTOCOLO`
- O tipo de protocolo é fundamental para a categorização e análise gerencial dos atendimentos

---
