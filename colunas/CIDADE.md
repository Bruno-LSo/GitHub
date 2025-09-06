# CIDADE - Cidade de Origem

## Descrição Geral

A coluna `CIDADE` identifica a localidade associada à unidade do funcionário responsável pelo protocolo, permitindo análise geográfica do atendimento.

## Origem dos Dados

### Tabelas Principais
- **Tabela**: `DB2INET.TB_LOCALIDADE`
- **Coluna de Origem**: `LOC_NO`
- **Tipo de Dado**: VARCHAR - Nome da localidade

### Obtenção através de CTE e JOINs

```sql
-- CTE UNIDADE_LOCALIDADE:
UNIDADE_LOCALIDADE AS (
    SELECT 
        U.DESC_UNI,
        U.DESC_SIGLA,
        U.CD_UNI,
        L.LOC_NO,
        ROW_NUMBER() OVER (PARTITION BY U.DESC_UNI, U.DESC_SIGLA ORDER BY L.LOC_NU) AS RN
    FROM DB2INET.TB_UNIDADE U
    INNER JOIN DB2INET.TB_UNIDADE_LOCALIDADE UL ON UL.CD_UNI = U.CD_UNI
    INNER JOIN DB2INET.TB_LOCALIDADE L ON L.LOC_NU = UL.LOC_NU
)

-- Na CTE PROTOCOLO_COMPLETO:
LEFT JOIN UNIDADE_LOCALIDADE UL 
    ON (UL.DESC_UNI = F.DEPARTMENT OR UL.DESC_SIGLA = F.DEPARTMENT) 
    AND UL.RN = 1

-- No SELECT final:
UL.LOC_NO AS CIDADE
```

## Lógica de Obtenção

### 1. Mapeamento Unidade → Localidade
A CTE cria o relacionamento:
- TB_UNIDADE → TB_UNIDADE_LOCALIDADE → TB_LOCALIDADE

### 2. Priorização com ROW_NUMBER
```sql
ROW_NUMBER() OVER (PARTITION BY U.DESC_UNI, U.DESC_SIGLA ORDER BY L.LOC_NU) AS RN
```
Garante uma única localidade por unidade (RN = 1)

### 3. Matching por Nome ou Sigla
```sql
ON (UL.DESC_UNI = F.DEPARTMENT OR UL.DESC_SIGLA = F.DEPARTMENT)
```
Busca tanto pelo nome completo quanto pela sigla da unidade

## Exemplos de Localidades

Baseando-se na tabela TB_LOCALIDADE:

| LOC_NO | UFE_SG | CEP |
|--------|--------|-----|
| Marquês dos Reis | PR | 86409000 |
| Palestina | CE | 63744000 |
| São Raimundo | CE | 63746000 |
| Lagoa da Cruz | CE | 62395000 |

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

## Exemplos Práticos no Resultado

Dados reais mostrando cidades:

| NR_PROT | UNIDADE | CIDADE |
|---------|---------|--------|
| PR0309572023 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | (null) |
| PR0310732023 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | (null) |
| PR0311262023 | GRE10 ARARAQUARA | (null) |
| PR0311962023 | GRE10 ARARAQUARA | (null) |

## Casos de Valor Nulo

A CIDADE fica nula quando:

1. **Unidade sem localidade mapeada**:
- Nem todas as unidades têm localidade associada em TB_UNIDADE_LOCALIDADE

2. **Protocolo sem atribuição**:
- Quando não há funcionário atribuído (DEPARTMENT nulo)

3. **Falha no matching**:
- Quando DEPARTMENT não corresponde a DESC_UNI ou DESC_SIGLA

## Relacionamento com Estrutura Organizacional

### Hierarquia de Dados
1. Protocolo → Funcionário (ATRIBUIDO)
2. Funcionário → Departamento (DEPARTMENT)
3. Departamento → Unidade (TB_UNIDADE)
4. Unidade → Localidade (TB_UNIDADE_LOCALIDADE)
5. Localidade → Cidade (TB_LOCALIDADE)

## Observações Técnicas

- Campo opcional frequentemente nulo nos dados
- Depende de múltiplos relacionamentos em cascata
- ROW_NUMBER() garante determinismo quando unidade tem múltiplas localidades
- Usado para análises geográficas e distribuição territorial
- A ordem por LOC_NU determina qual localidade é escolhida quando há múltiplas
- Importante para relatórios de cobertura geográfica do atendimento