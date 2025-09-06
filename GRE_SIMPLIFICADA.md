# GRE_SIMPLIFICADA - GRE Simplificada para Análise

## Descrição Geral

A coluna `GRE_SIMPLIFICADA` consolida e padroniza as identificações de GRE e departamentos em categorias simplificadas, facilitando análises gerenciais e relatórios consolidados.

## Origem dos Dados

### Cálculo Baseado em Mapeamento e Simplificação
- **Tipo de Dado**: VARCHAR - Categoria simplificada
- **Base**: Transformação da coluna GRE com padronizações

## Regra Principal de Simplificação

```sql
-- No SELECT final (trecho principal):
CASE
    -- Regra 1: Cancelado por não pagamento
    WHEN DS_STATUS = 'Solicitação Cancelada' 
         AND FINISHED = 1 
         AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
    THEN 'Cancelado por não pagamento'
    
    -- Regra 2: Funcionário com Acervo
    WHEN TEM_ACERVO IS NOT NULL 
    THEN 'ACERVO'
    
    -- Regra 3: Sem distribuição
    WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NOT NULL
        AND COALESCE(DEPARTMENT, '') = ''
        AND COALESCE(GRE_DESC, '') = ''
        AND COALESCE(CIDADE, '') = ''
        AND COALESCE(ATRIBUIDO, '') = ''
    THEN 'Sem distribuição'
    
    -- Regra 4: Aguardando Pagamento vira Sem distribuição
    WHEN DS_STATUS = 'Aguardando Pagamento' 
    THEN 'Sem distribuição'
    
    -- Regra 5: GREs padrão (GRE1 a GRE12)
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE5' THEN 'GRE5'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE12' THEN 'GRE12'
    -- ... (todas as 12 GREs)
    
    -- Regra 6: Departamentos especiais
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO' 
    THEN 'ACERVO'  -- CORRIGIDO: Era 'ATENDIMENTO', agora é 'ACERVO'
```

## Mapeamentos de Departamentos

### Departamentos Transformados em Siglas

| Departamento Original | GRE_SIMPLIFICADA |
|----------------------|------------------|
| EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO |
| EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS | ADM DADOS |
| UPS SANTOS | UPS SANTOS |
| UNIDADE CAMARAS ESPECIALIZADAS E CONSULTAS TECNICA | CONSULTA TÉCNICA |
| GABINETE DA PRESIDENCIA | PRESIDÊNCIA |
| UNIDADE FINANCEIRA E CONTABIL - UFC | UFC |
| UNIDADE DE PARCERIA E RELACOES INSTITUCIONAIS - UR | UR |
| EQUIPE DE SERV ACERVO TECNICO E OPERACIONAL, PROTO | ACERVO TÉCNICO |
| GERENCIA DE EXPERIENCIA E JORNADA - GEJ | GEJ |
| GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC | GFISC |
| UGI REGISTRO | UGI REGISTRO |
| EQUIPE DE OPERACAO, QUALIDADE E SUCESSO DO CLIENTE | EQUIPE OPERAÇÃO |
| GERENCIA DE ASSUNTOS JURIDICOS | JURÍDICO |
| GERENCIA DE DES. E EXECUCAO DE PROJETOS - GDEP | GDEP |

## Casos Especiais

### 1. Transformação ATENDIMENTO → ACERVO
```sql
-- IMPORTANTE: Correção aplicada
WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO' 
THEN 'ACERVO'  -- CORRIGIDO: Era 'ATENDIMENTO', agora é 'ACERVO'
```

### 2. Status Especiais
```sql
-- Sem pagamento após prazo
WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
     AND DT_PAGAMENTO IS NULL 
     AND NR_BOLETO IS NOT NULL
THEN 'CANCELADO SEM PAGTO'
```

### 3. Casos Não Atribuídos
```sql
WHEN UPPER(TRIM(COALESCE(GRE_FROM_MAPPING, DEPARTMENT, ''))) IN ('NÃO ATRIBUÍDO', 'NAO ATRIBUIDO') 
THEN 'Sem distribuição'
```

## Exemplos Práticos no Resultado

Dados reais mostrando simplificações:

| NR_PROT | GRE | GRE_SIMPLIFICADA |
|---------|-----|------------------|
| PR0309572023 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO |
| PR0310732023 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO |
| PR0311262023 | GRE10 | GRE10 |
| PR0311962023 | GRE10 | GRE10 |

## Filtros Aplicados no WHERE

A query exclui determinadas GREs simplificadas:

```sql
WHERE ... NOT IN (
    'SEM CLASSIFICAÇÃO',
    'PRESIDÊNCIA',
    'EISI',
    'GDEP',
    'GFISC',
    'ADM DADOS',
    'CONSULTA TÉCNICA',
    'EQUIPE OPERAÇÃO',
    'GEJ',
    'JURÍDICO'
)
```

Também exclui com LIKE:
```sql
AND UPPER(...) NOT LIKE '%ADM DE DADOS%'
AND UPPER(...) NOT LIKE '%CONSULTA TÉCNICA%'
AND UPPER(...) NOT LIKE '%EQUIPE OPERAÇÃO%'
AND UPPER(...) NOT LIKE '%GEJ%'
AND UPPER(...) NOT LIKE '%GDEP%'
AND UPPER(...) NOT LIKE '%GFISC%'
```

## Observações Técnicas

- Simplifica nomes longos em siglas padronizadas
- Agrupa variações do mesmo departamento
- Facilita criação de dashboards e relatórios
- Reduz complexidade para usuários finais
- Aplicação de filtros garante foco em áreas operacionais principais
- Correção importante: ATENDIMENTO foi unificado com ACERVO