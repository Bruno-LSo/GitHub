# REGIONAL - Regional do CREA-SP

## Descrição Geral

A coluna `REGIONAL` identifica a Gerência Regional (GRE) associada ao departamento do funcionário responsável, fornecendo uma visão regionalizada do atendimento.

## Origem dos Dados

### Tabela/View Principal
- **View**: `DB2INET.VW_UNIDADES_GRES`
- **Coluna de Origem**: `GRE_DESC`
- **Tipo de Dado**: VARCHAR - Descrição da GRE

### Obtenção através de JOIN

```sql
-- Na CTE PROTOCOLO_COMPLETO:
LEFT JOIN DB2INET.VW_UNIDADES_GRES GRES ON GRES.DESC_UNI = F.DEPARTMENT

-- No SELECT final:
GRE_DESC AS REGIONAL
```

## Estrutura da View VW_UNIDADES_GRES

Baseando-se nos dados da view:

| DESC_UNI | SIGLA_UNI | GRE_SIGLA | GRE_DESC |
|----------|-----------|-----------|----------|
| UPS AEASP | UPSAEASP | GRE5 | GRE5 CAPITAL |
| UPS IBAPE | UPSIBAPE | GRE5 | GRE5 CAPITAL |
| UGI PIRACICABA | UGIPIRA | GRE4 | GRE4 PIRACICABA |

## Exemplos Práticos no Resultado

Dados reais mostrando a regional:

| NR_PROT | UNIDADE | REGIONAL | GRE_SIGLA |
|---------|---------|----------|-----------|
| PR0309572023 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | (null) | - |
| PR0310732023 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | (null) | - |
| PR0311262023 | GRE10 ARARAQUARA | GRE10 ARARAQUARA | GRE10 |
| PR0311962023 | GRE10 ARARAQUARA | GRE10 ARARAQUARA | GRE10 |

## Casos de Valor Nulo

A REGIONAL fica nula quando:

1. **Unidade não mapeada na view**:
```sql
-- Exemplo: "EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO" 
-- não existe em VW_UNIDADES_GRES.DESC_UNI
```

2. **Protocolo sem atribuição**:
```sql
-- Quando DEPARTMENT é nulo ou vazio
AND COALESCE(GRE_DESC, '') = ''
```

## Utilização nas Regras de Negócio

### Identificação de Protocolos Sem Distribuição

```sql
-- Na coluna GRE:
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NOT NULL
    AND COALESCE(DEPARTMENT, '') = ''
    AND COALESCE(GRE_DESC, '') = ''
    AND COALESCE(CIDADE, '') = ''
    AND COALESCE(ATRIBUIDO, '') = ''
THEN 'Sem distribuição'
```

## Relacionamento com GRE_FROM_MAPPING

A query utiliza uma CTE alternativa para mapear GREs:

```sql
-- CTE GRE_MAPPING usa uma lógica diferente:
SELECT VW_GRE.GRE_SIGLA
FROM DB2INET.TB_FUNCIONARIO F
INNER JOIN DB2INET.TB_UNIDADE U 
    ON (U.DESC_UNI = F.DEPARTMENT OR U.DESC_SIGLA = F.DEPARTMENT)
INNER JOIN DB2INET.VW_UNIDADES_GRES VW_GRE 
    ON U.DESC_SIGLA = VW_GRE.SIGLA_UNI
```

Esta abordagem alternativa captura GREs através da sigla da unidade.

## GREs Identificadas no Sistema

Baseando-se nos dados e na query:

| GRE | Descrição |
|-----|-----------|
| GRE1 | Regional 1 |
| GRE2 | Regional 2 |
| GRE3 | Regional 3 |
| GRE4 | GRE4 PIRACICABA |
| GRE5 | GRE5 CAPITAL |
| GRE6 | Regional 6 |
| GRE7 | Regional 7 |
| GRE8 | Regional 8 |
| GRE9 | Regional 9 |
| GRE10 | GRE10 ARARAQUARA |
| GRE11 | Regional 11 |
| GRE12 | Regional 12 |

## Observações Técnicas

- Campo opcional que pode ser nulo
- Depende do mapeamento correto em VW_UNIDADES_GRES
- Usado para análises regionalizadas de atendimento
- Nem todas as unidades possuem regional associada
- A view VW_UNIDADES_GRES contém hierarquia (NVL_HIERARQUIA) e número da GRE (NGRE)

---

