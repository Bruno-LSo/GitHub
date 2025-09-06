# UNIDADE - Unidade Organizacional

## Descrição Geral

A coluna `UNIDADE` identifica o departamento ou unidade organizacional do funcionário responsável pelo protocolo, refletindo a estrutura organizacional do CREA-SP.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_FUNCIONARIO`
- **Coluna de Origem**: `DEPARTMENT`
- **Tipo de Dado**: VARCHAR - Nome do departamento/unidade

### Obtenção através de JOIN

```sql
-- Na CTE PROTOCOLO_COMPLETO:
LEFT JOIN DB2INET.TB_FUNCIONARIO F 
    ON UPPER(F.SAMACCOUNTNAME) = UPPER(US.ATRIBUIDO) 
    AND F.DELETED = 0

-- No SELECT final:
DEPARTMENT AS UNIDADE
```

## Valores Encontrados

Exemplos de unidades baseados nos dados:

| UNIDADE | Descrição |
|---------|-----------|
| EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | Equipe de atendimento principal |
| GRE10 ARARAQUARA | Gerência Regional 10 - Araraquara |
| GERENCIA DE EXPERIENCIA E JORNADA - GEJ | Gerência de experiência |
| GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC | Gerência de fiscalização |
| UPS SANTOS | Unidade de Pronto Atendimento Santos |

## Utilização nas Regras de Negócio

### 1. Determinação da GRE

A unidade é fundamental para determinar a GRE do protocolo:

```sql
-- JOIN com VW_UNIDADES_GRES:
LEFT JOIN DB2INET.VW_UNIDADES_GRES GRES ON GRES.DESC_UNI = F.DEPARTMENT
```

### 2. Mapeamento Alternativo de Unidades

A query também verifica pela sigla da unidade:

```sql
-- Na CTE GRE_MAPPING:
INNER JOIN DB2INET.TB_UNIDADE U 
    ON (U.DESC_UNI = F.DEPARTMENT OR U.DESC_SIGLA = F.DEPARTMENT)
```

### 3. Identificação de Protocolos Sem Distribuição

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

Dados reais mostrando unidades:

| NR_PROT | ATRIBUIDO | UNIDADE | REGIONAL |
|---------|-----------|---------|----------|
| PR0309572023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | (null) |
| PR0310732023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | (null) |
| PR0311262023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 ARARAQUARA |
| PR0311962023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 ARARAQUARA |

## Mapeamento para GRE_Simplificada

A unidade influencia diretamente a classificação simplificada:

```sql
-- Transformações específicas por departamento:
WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO' 
THEN 'ACERVO'

WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS' 
THEN 'ADM DADOS'

WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'UPS SANTOS' 
THEN 'UPS SANTOS'
```

## Relacionamento com Outras Colunas

### REGIONAL
- Se DEPARTMENT existe em VW_UNIDADES_GRES.DESC_UNI → preenche REGIONAL com GRE_DESC
- Se não → REGIONAL fica nulo

### CIDADE
Através da CTE UNIDADE_LOCALIDADE:
```sql
SELECT U.DESC_UNI, L.LOC_NO
FROM DB2INET.TB_UNIDADE U
INNER JOIN DB2INET.TB_UNIDADE_LOCALIDADE UL ON UL.CD_UNI = U.CD_UNI
INNER JOIN DB2INET.TB_LOCALIDADE L ON L.LOC_NU = UL.LOC_NU
```

### GRE
Utiliza DEPARTMENT como fallback quando não há mapeamento de GRE:
```sql
ELSE COALESCE(GRE_FROM_MAPPING, DEPARTMENT)
```

## Observações Técnicas

- Campo pode ser nulo quando protocolo não está atribuído
- Reflete a estrutura organizacional atual do CREA-SP
- Usado para análises de distribuição de carga por departamento
- Fundamental para relatórios gerenciais por unidade
- Algumas unidades têm nomes longos que podem ser truncados em relatórios

---
