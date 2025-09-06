# ATRIBUIDO - Funcionário Responsável

## Descrição Geral

A coluna `ATRIBUIDO` identifica o funcionário responsável pelo protocolo, contendo o login (SAMACCOUNTNAME) do último agente que processou ou está processando a solicitação.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_STATUS_PROTOCOLO_CREANET`
- **Coluna de Origem**: `ATRIBUIDO`
- **Tipo de Dado**: VARCHAR - Login do funcionário

### Obtenção através da CTE

```sql
-- Na CTE ULTIMO_STATUS:
SELECT 
    SP.CD_PROT,
    SP.ATRIBUIDO,
    -- outras colunas...
    ROW_NUMBER() OVER (PARTITION BY SP.CD_PROT ORDER BY SP.CD_STAT_PROT DESC) AS RN
FROM DB2INET.TB_STATUS_PROTOCOLO_CREANET SP
```

## Utilização nas Regras de Negócio

### 1. Identificação de Funcionários com Acervo

A query identifica funcionários que trabalham com Acervo Técnico:

```sql
-- CTE FUNCIONARIOS_ACERVO:
SELECT DISTINCT 
    UPPER(SP.ATRIBUIDO) AS ATRIBUIDO_UPPER
FROM DB2INET.TB_PROTOCOLO_CREANET P
INNER JOIN DB2INET.TB_STATUS_PROTOCOLO_CREANET SP ON SP.CD_PROT = P.CD_PROT
WHERE P.CD_TP_PROT IN (
    SELECT CD_TP_PROT 
    FROM DB2INET.TB_TIPO_PROTOCOLO
    WHERE NM_TP_PROT = 'Acervo Técnico'
)
AND SP.ATRIBUIDO IS NOT NULL
AND LENGTH(TRIM(SP.ATRIBUIDO)) > 0
```

### 2. Relacionamento com Dados do Funcionário

```sql
-- JOIN para obter dados do funcionário:
LEFT JOIN DB2INET.TB_FUNCIONARIO F 
    ON UPPER(F.SAMACCOUNTNAME) = UPPER(US.ATRIBUIDO) 
    AND F.DELETED = 0
```

### 3. Validação de Atribuição

A query verifica se o protocolo está atribuído:

```sql
-- Na coluna GRE:
WHEN COALESCE(DS_OBSERVACOES, '') = '' 
    OR COALESCE(ATRIBUIDO, '') = ''
    OR ATRIBUIDO = '0'
THEN
    CASE
        WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
             AND DT_PAGAMENTO IS NULL 
             AND NR_BOLETO IS NOT NULL
        THEN 'Sem pagamento'
        ELSE 'Protocolo ainda não atribuído a um agente'
    END
```

## Exemplos Práticos no Resultado

Dados reais mostrando atribuições:

| NR_PROT | ATRIBUIDO | UNIDADE | GRE |
|---------|-----------|---------|-----|
| PR0309572023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO |
| PR0310732023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO |
| PR0311262023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 |
| PR0311962023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 |

## Casos Especiais de Atribuição

### Funcionário com Acervo
```sql
-- Verificação na coluna GRE:
WHEN TEM_ACERVO IS NOT NULL
THEN 'ACERVO'
```

### Sem Atribuição
Quando `ATRIBUIDO` está vazio, nulo ou igual a '0':
- GRE recebe: "Protocolo ainda não atribuído a um agente"
- GRE_Simplificada recebe: "Sem distribuição"

### Login Especial "0"
```sql
WHEN ATRIBUIDO = '0'
```
Tratado como não atribuído

## Relacionamento com Outras Colunas

### UNIDADE
Obtida através do JOIN com TB_FUNCIONARIO usando o campo DEPARTMENT

### REGIONAL
Obtida através do JOIN com VW_UNIDADES_GRES baseado no DEPARTMENT do funcionário

### GRE e GRE_SIMPLIFICADA
Determinadas com base no funcionário atribuído e sua unidade organizacional

## Mapeamento de GRE

A query mapeia funcionários para GREs através da CTE:

```sql
-- CTE GRE_MAPPING:
SELECT 
    F.SAMACCOUNTNAME,
    F.DEPARTMENT,
    UPPER(F.SAMACCOUNTNAME) AS SAM_UPPER,
    VW_GRE.GRE_SIGLA,
    ROW_NUMBER() OVER (PARTITION BY UPPER(F.SAMACCOUNTNAME) ORDER BY VW_GRE.GRE_SIGLA) AS RN
FROM DB2INET.TB_FUNCIONARIO F
INNER JOIN DB2INET.TB_UNIDADE U ON (U.DESC_UNI = F.DEPARTMENT OR U.DESC_SIGLA = F.DEPARTMENT)
INNER JOIN DB2INET.VW_UNIDADES_GRES VW_GRE ON U.DESC_SIGLA = VW_GRE.SIGLA_UNI
WHERE COALESCE(F.DELETED, 0) = 0
```

## Observações Técnicas

- O campo é case-insensitive nas comparações (usa UPPER())
- Pode conter valores nulos ou vazios indicando não atribuição
- O valor "0" é tratado especialmente como não atribuído
- É chave para rastreabilidade e auditoria de responsabilidades
- Fundamental para distribuição de carga de trabalho entre equipes

---
