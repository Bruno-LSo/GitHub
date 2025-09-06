# GRE - Gerência Regional

## Descrição Geral

A coluna `GRE` apresenta a identificação da Gerência Regional responsável pelo protocolo, com regras complexas que incluem validações de pagamento, atribuição, situações especiais e mapeamentos departamentais.

## Origem dos Dados

### Cálculo Complexo Baseado em Múltiplas Tabelas
- **Tipo de Dado**: VARCHAR - Identificação da GRE ou status especial
- **Tabelas de Origem**:
  - `DB2INET.TB_FUNCIONARIO`: DEPARTMENT, SAMACCOUNTNAME
  - `DB2INET.VW_UNIDADES_GRES`: GRE_DESC, GRE_SIGLA
  - `DB2INET.TB_UNIDADE`: DESC_UNI, DESC_SIGLA
  - `DB2INET.TB_STATUS_PROTOCOLO_CREANET`: ATRIBUIDO, DS_OBSERVACOES
  - `DB2INET.TB_PROTOCOLO_CREANET`: Para identificação de Acervo Técnico
  - `DB2INET.TB_BOLETO`: NR_BOLETO, DT_PAGAMENTO

## Regra Principal Completa de Determinação

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
    
    -- Regra 3: Sem distribuição (pago mas sem atribuição completa)
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

## CTEs que Alimentam a Coluna GRE

### CTE FUNCIONARIOS_ACERVO

Identifica funcionários que trabalham com Acervo Técnico:

```sql
FUNCIONARIOS_ACERVO AS (
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
    AND P.DRAFT = 0
)
```

### CTE GRE_MAPPING

Mapeia funcionários para suas respectivas GREs:

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

## Detalhamento de Cada Regra

### Regra 1: Cancelado por não pagamento (Prioridade Máxima)

```sql
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 'Cancelado por não pagamento'
```

**Exemplo Real**:
| NR_PROT | DS_STATUS | FINISHED | DS_OBSERVACOES | GRE |
|---------|-----------|----------|----------------|-----|
| PR0311262023 | Solicitação Cancelada | SIM | Boleto vencido: 28027180231412766 - Vencto.: 04/07/2023 00:00:00 | Cancelado por não pagamento |
| PR0311962023 | Solicitação Cancelada | SIM | Boleto vencido: 28027180231414609 - Vencto.: 04/07/2023 00:00:00 | Cancelado por não pagamento |

### Regra 2: ACERVO

```sql
WHEN TEM_ACERVO IS NOT NULL
THEN 'ACERVO'
```

**Condição**: Funcionário identificado na CTE FUNCIONARIOS_ACERVO (trabalhou com Acervo Técnico)

**Exemplo Real**:
| NR_PROT | ATRIBUIDO | DEPARTMENT | TEM_ACERVO | GRE |
|---------|-----------|------------|------------|-----|
| PR0309572023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | rosana.torre4031 | ACERVO |
| PR0310732023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | rosana.torre4031 | ACERVO |

### Regra 3: Sem distribuição (Pago mas sem atribuição)

```sql
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NOT NULL
    AND COALESCE(DEPARTMENT, '') = ''
    AND COALESCE(GRE_DESC, '') = ''
    AND COALESCE(CIDADE, '') = ''
    AND COALESCE(ATRIBUIDO, '') = ''
THEN 'Sem distribuição'
```

**Condições**:
- Boleto emitido e pago
- Mas sem departamento, GRE, cidade ou funcionário atribuído

### Regra 4: Não atribuído ou Login especial

```sql
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

**Subregra 4.1**: "Sem pagamento" quando vencido sem pagamento
**Subregra 4.2**: "Protocolo ainda não atribuído a um agente" nos demais casos

### Regra 5: Usa mapeamento GRE ou departamento (ELSE)

```sql
ELSE COALESCE(GRE_FROM_MAPPING, DEPARTMENT)
```

**Lógica COALESCE**:
1. Primeiro tenta usar GRE_FROM_MAPPING (da CTE GRE_MAPPING)
2. Se NULL, usa DEPARTMENT (do funcionário)

**Exemplos Reais de Mapeamento**:

| NR_PROT | ATRIBUIDO | DEPARTMENT | GRE_FROM_MAPPING | GRE (resultado) |
|---------|-----------|------------|------------------|-----------------|
| PR0311262023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 | GRE10 |
| PR0311962023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 | GRE10 |

## Valores Possíveis de GRE

### Valores Especiais (Prioridade Alta)
- **"Cancelado por não pagamento"** - Protocolos cancelados com boleto vencido
- **"ACERVO"** - Funcionários que trabalham com Acervo Técnico
- **"Sem distribuição"** - Protocolos pagos mas não atribuídos
- **"Sem pagamento"** - Protocolos vencidos com boleto não pago
- **"Protocolo ainda não atribuído a um agente"** - Sem atribuição

### GREs Regionais (GRE1 a GRE12)
Quando há mapeamento via GRE_MAPPING:
- GRE1, GRE2, GRE3, GRE4, GRE5, GRE6, GRE7, GRE8, GRE9, GRE10, GRE11, GRE12

### Departamentos (quando não há mapeamento GRE)
Quando COALESCE usa DEPARTMENT diretamente:

| Departamento | Aparece como |
|--------------|--------------|
| EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO |
| EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS | EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS |
| UPS SANTOS | UPS SANTOS |
| UNIDADE CAMARAS ESPECIALIZADAS E CONSULTAS TECNICA | UNIDADE CAMARAS ESPECIALIZADAS E CONSULTAS TECNICA |
| GABINETE DA PRESIDENCIA | GABINETE DA PRESIDENCIA |
| UNIDADE FINANCEIRA E CONTABIL - UFC | UNIDADE FINANCEIRA E CONTABIL - UFC |
| UNIDADE DE PARCERIA E RELACOES INSTITUCIONAIS - UR | UNIDADE DE PARCERIA E RELACOES INSTITUCIONAIS - UR |
| EQUIPE DE SERV ACERVO TECNICO E OPERACIONAL, PROTO | EQUIPE DE SERV ACERVO TECNICO E OPERACIONAL, PROTO |
| GERENCIA DE EXPERIENCIA E JORNADA - GEJ | GERENCIA DE EXPERIENCIA E JORNADA - GEJ |
| GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC | GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC |
| UGI REGISTRO | UGI REGISTRO |
| EISI | EISI |
| EQUIPE DE OPERACAO, QUALIDADE E SUCESSO DO CLIENTE | EQUIPE DE OPERACAO, QUALIDADE E SUCESSO DO CLIENTE |
| GERENCIA DE ASSUNTOS JURIDICOS | GERENCIA DE ASSUNTOS JURIDICOS |
| GERENCIA DE DES. E EXECUCAO DE PROJETOS - GDEP | GERENCIA DE DES. E EXECUCAO DE PROJETOS - GDEP |

## JOINs que Alimentam a Coluna

```sql
-- Obtém departamento do funcionário
LEFT JOIN DB2INET.TB_FUNCIONARIO F 
    ON UPPER(F.SAMACCOUNTNAME) = UPPER(US.ATRIBUIDO) 
    AND F.DELETED = 0

-- Obtém GRE_DESC da view
LEFT JOIN DB2INET.VW_UNIDADES_GRES GRES 
    ON GRES.DESC_UNI = F.DEPARTMENT

-- Obtém mapeamento GRE via CTE
LEFT JOIN GRE_MAPPING GM 
    ON GM.SAM_UPPER = UPPER(US.ATRIBUIDO) 
    AND GM.RN = 1

-- Verifica se funcionário tem acervo
LEFT JOIN FUNCIONARIOS_ACERVO FA 
    ON FA.ATRIBUIDO_UPPER = UPPER(US.ATRIBUIDO)
```

## Hierarquia de Prioridades

A ordem de avaliação é crucial:

```
1º - Cancelado por não pagamento (máxima prioridade)
2º - ACERVO (funcionário com histórico de Acervo Técnico)
3º - Sem distribuição (pago mas sem atribuição completa)
4º - Não atribuído (com subcasos "Sem pagamento" ou "Protocolo ainda não atribuído")
5º - Mapeamento GRE ou Departamento (menor prioridade)
```

## Relacionamento com GRE_Simplificada

A coluna GRE é a base para GRE_Simplificada, que aplica simplificações adicionais:
- Departamentos longos → Siglas
- "Aguardando Pagamento" → "Sem distribuição"
- Padronizações diversas

## Observações Técnicas

- Campo complexo com múltiplas origens possíveis
- Usa COALESCE extensivamente para tratar NULLs
- UPPER() usado para comparações case-insensitive de logins
- "ACERVO" tem precedência sobre mapeamento regional
- Valor "0" em ATRIBUIDO é tratado como não atribuído
- Fundamental para distribuição e análise de carga de trabalho
- Permite identificação de protocolos problemáticos ou não distribuídos

---