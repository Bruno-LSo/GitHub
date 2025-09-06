# GRE - Gerência Regional

## Descrição Geral

A coluna `GRE` apresenta a identificação da Gerência Regional responsável pelo protocolo, com regras complexas que incluem validações de pagamento, atribuição e situações especiais.

## Origem dos Dados

### Cálculo Complexo Baseado em Múltiplas Condições
- **Tipo de Dado**: VARCHAR - Identificação da GRE ou status especial
- **Base**: Combinação de atribuição, pagamento, departamento e mapeamentos

## Regra Principal de Determinação

```sql
-- No SELECT final:
CASE
    -- Regra 1: Cancelado por não pagamento
    WHEN DS_STATUS = 'Solicitação Cancelada' 
         AND FINISHED = 1 
         AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
    THEN 'Cancelado por não pagamento'
    
    -- Regra 2: Funcionário com Acervo
    WHEN TEM_ACERVO IS NOT NULL
    THEN 'ACERVO'
    
    -- Regra 3: Sem distribuição (pago mas sem atribuição)
    WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NOT NULL
        AND COALESCE(DEPARTMENT, '') = ''
        AND COALESCE(GRE_DESC, '') = ''
        AND COALESCE(CIDADE, '') = ''
        AND COALESCE(ATRIBUIDO, '') = ''
    THEN 'Sem distribuição'
    
    -- Regra 4: Não atribuído ou Login = 0
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
    
    -- Regra 5: Usa mapeamento ou departamento
    ELSE COALESCE(GRE_FROM_MAPPING, DEPARTMENT)
END AS GRE
```

## Casos Especiais de GRE

### 1. Cancelado por não pagamento
**Condições**: Cancelado + Finalizado + "Boleto Vencido" nas observações

**Exemplo Real**:
| NR_PROT | DS_OBSERVACOES | GRE |
|---------|----------------|-----|
| PR0311262023 | Boleto vencido: 28027180231412766... | Cancelado por não pagamento |

### 2. ACERVO
**Condição**: Funcionário identificado na CTE FUNCIONARIOS_ACERVO

```sql
-- CTE FUNCIONARIOS_ACERVO identifica funcionários que trabalham com Acervo Técnico
LEFT JOIN FUNCIONARIOS_ACERVO FA ON FA.ATRIBUIDO_UPPER = UPPER(US.ATRIBUIDO)
```

**Exemplo Real**:
| NR_PROT | ATRIBUIDO | UNIDADE | GRE |
|---------|-----------|---------|-----|
| PR0309572023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO |

### 3. Sem distribuição
**Condições**: Boleto pago mas sem atribuição completa

### 4. Protocolo ainda não atribuído a um agente
**Condições**: Sem observações, sem atribuído ou atribuído = '0'

### 5. Sem pagamento
**Condições**: Vencido + Sem pagamento + Com boleto

## Mapeamento de GRE

A CTE GRE_MAPPING cria o mapeamento:

```sql
GRE_MAPPING AS (
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
)
```

## Exemplos Práticos no Resultado

Dados reais mostrando diferentes GREs:

| NR_PROT | ATRIBUIDO | DEPARTMENT | GRE_FROM_MAPPING | GRE |
|---------|-----------|------------|------------------|-----|
| PR0309572023 | rosana.torre4031 | EQUIPE DE ATEND... | NULL | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO |
| PR0310732023 | rosana.torre4031 | EQUIPE DE ATEND... | NULL | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO |
| PR0311262023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 | GRE10 |
| PR0311962023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 | GRE10 |

## Fallback para Department

Quando não há mapeamento específico:
```sql
ELSE COALESCE(GRE_FROM_MAPPING, DEPARTMENT)
```
Usa o departamento como GRE se não houver mapeamento

## Observações Técnicas

- Campo complexo com múltiplas origens possíveis
- Prioriza situações especiais sobre mapeamento normal
- "ACERVO" tem precedência sobre GRE regional
- Fundamental para distribuição e análise de carga de trabalho
- Permite identificação de protocolos problemáticos

---
