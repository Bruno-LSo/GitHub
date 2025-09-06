# GRE_SIMPLIFICADA - GRE Simplificada para Análise

## Descrição Geral

A coluna `GRE_SIMPLIFICADA` consolida e padroniza as identificações de GRE e departamentos em categorias simplificadas, facilitando análises gerenciais e relatórios consolidados. Aplica transformações e filtros específicos para o BI do atendimento.

## Origem dos Dados

### Cálculo Baseado em Mapeamento e Simplificação
- **Tipo de Dado**: VARCHAR - Categoria simplificada
- **Tabelas de Origem**:
  - `DB2INET.TB_FUNCIONARIO`: DEPARTMENT, SAMACCOUNTNAME
  - `DB2INET.VW_UNIDADES_GRES`: GRE_SIGLA (via CTE GRE_MAPPING)
  - `DB2INET.TB_UNIDADE`: DESC_UNI, DESC_SIGLA
  - `DB2INET.TB_STATUS_PROTOCOLO_CREANET`: ATRIBUIDO, DS_OBSERVACOES
  - `DB2INET.TB_PROTOCOLO_CREANET`: Para identificação de Acervo
  - `DB2INET.TB_BOLETO`: NR_BOLETO, DT_PAGAMENTO
  - `DB2INET.TB_TIPO_STATUS_PROTOCOLO`: DS_STATUS

## Regra Completa de Simplificação

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
    
    -- Regra 4: Aguardando Pagamento vira Sem distribuição
    WHEN DS_STATUS = 'Aguardando Pagamento' 
    THEN 'Sem distribuição'
    
    -- Regra 5: GREs padrão (mantém nome)
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE5' THEN 'GRE5'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE12' THEN 'GRE12'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE2' THEN 'GRE2'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE7' THEN 'GRE7'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE9' THEN 'GRE9'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE10' THEN 'GRE10'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE11' THEN 'GRE11'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE3' THEN 'GRE3'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE4' THEN 'GRE4'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE1' THEN 'GRE1'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE8' THEN 'GRE8'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GRE6' THEN 'GRE6'
    
    -- Regra 6: Status especiais - não atribuído
    WHEN COALESCE(DS_OBSERVACOES, '') = '' 
        OR COALESCE(ATRIBUIDO, '') = ''
        OR ATRIBUIDO = '0'
    THEN 'Sem distribuição'
    
    -- Regra 7: Cancelado sem pagamento
    WHEN DAYS_CURRENT > DAYS(DT_EST_CONCLUSAO) 
         AND DT_PAGAMENTO IS NULL 
         AND NR_BOLETO IS NOT NULL
    THEN 'CANCELADO SEM PAGTO'
    
    -- Regra 8: Departamentos com simplificação
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO' 
    THEN 'ACERVO'  -- CORRIGIDO: Era 'ATENDIMENTO', agora é 'ACERVO'
    
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS' THEN 'ADM DADOS'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'UPS SANTOS' THEN 'UPS SANTOS'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'UNIDADE CAMARAS ESPECIALIZADAS E CONSULTAS TECNICA' THEN 'CONSULTA TÉCNICA'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GABINETE DA PRESIDENCIA' THEN 'PRESIDÊNCIA'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'UNIDADE FINANCEIRA E CONTABIL - UFC' THEN 'UFC'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'UNIDADE DE PARCERIA E RELACOES INSTITUCIONAIS - UR' THEN 'UR'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) IN ('EQUIPE DE SERV ACERVO TECNICO E OPERACIONAL, PROTO',
                                                      'EQUIPE DE SERV ACERVO TÉCNICO E OPERACIONAL, PROTO') THEN 'ACERVO TÉCNICO'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GERENCIA DE EXPERIENCIA E JORNADA - GEJ' THEN 'GEJ'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC' THEN 'GFISC'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'UGI REGISTRO' THEN 'UGI REGISTRO'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EISI' THEN 'EISI'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE OPERACAO, QUALIDADE E SUCESSO DO CLIENTE' THEN 'EQUIPE OPERAÇÃO'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GERENCIA DE ASSUNTOS JURIDICOS' THEN 'JURÍDICO'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'GERENCIA DE DES. E EXECUCAO DE PROJETOS - GDEP' THEN 'GDEP'
    
    -- Regra 9: Casos especiais com UPPER/TRIM
    WHEN UPPER(TRIM(COALESCE(GRE_FROM_MAPPING, DEPARTMENT, ''))) IN ('NÃO ATRIBUÍDO', 'NAO ATRIBUIDO') THEN 'Sem distribuição'
    WHEN UPPER(TRIM(COALESCE(GRE_FROM_MAPPING, DEPARTMENT, ''))) IN ('SEM CLASSIFICAÇÃO', 'SEM CLASSIFICACAO') THEN 'SEM CLASSIFICAÇÃO'
    WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT, '') = '' THEN 'Sem distribuição'
    
    -- Regra 10: ELSE - mantém valor original
    ELSE COALESCE(GRE_FROM_MAPPING, DEPARTMENT)
END AS GRE_Simplificada
```

## Tabela Completa de Mapeamentos

### Casos Especiais (Prioridade Alta)

| Condição | GRE_SIMPLIFICADA |
|----------|------------------|
| Cancelado com boleto vencido | Cancelado por não pagamento |
| Funcionário com Acervo (TEM_ACERVO NOT NULL) | ACERVO |
| Pago mas sem atribuição completa | Sem distribuição |
| DS_STATUS = 'Aguardando Pagamento' | Sem distribuição |
| Não atribuído ou ATRIBUIDO = '0' | Sem distribuição |
| Vencido sem pagamento com boleto | CANCELADO SEM PAGTO |

### GREs Regionais (Mantém Nome Original)

| GRE Original | GRE_SIMPLIFICADA |
|--------------|------------------|
| GRE1 | GRE1 |
| GRE2 | GRE2 |
| GRE3 | GRE3 |
| GRE4 | GRE4 |
| GRE5 | GRE5 |
| GRE6 | GRE6 |
| GRE7 | GRE7 |
| GRE8 | GRE8 |
| GRE9 | GRE9 |
| GRE10 | GRE10 |
| GRE11 | GRE11 |
| GRE12 | GRE12 |

### Departamentos Transformados

| Departamento Original | GRE_SIMPLIFICADA |
|----------------------|------------------|
| EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | **ACERVO** |
| EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS | ADM DADOS |
| UPS SANTOS | UPS SANTOS |
| UNIDADE CAMARAS ESPECIALIZADAS E CONSULTAS TECNICA | CONSULTA TÉCNICA |
| GABINETE DA PRESIDENCIA | PRESIDÊNCIA |
| UNIDADE FINANCEIRA E CONTABIL - UFC | UFC |
| UNIDADE DE PARCERIA E RELACOES INSTITUCIONAIS - UR | UR |
| EQUIPE DE SERV ACERVO TECNICO E OPERACIONAL, PROTO | ACERVO TÉCNICO |
| EQUIPE DE SERV ACERVO TÉCNICO E OPERACIONAL, PROTO | ACERVO TÉCNICO |
| GERENCIA DE EXPERIENCIA E JORNADA - GEJ | GEJ |
| GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC | GFISC |
| UGI REGISTRO | UGI REGISTRO |
| EISI | EISI |
| EQUIPE DE OPERACAO, QUALIDADE E SUCESSO DO CLIENTE | EQUIPE OPERAÇÃO |
| GERENCIA DE ASSUNTOS JURIDICOS | JURÍDICO |
| GERENCIA DE DES. E EXECUCAO DE PROJETOS - GDEP | GDEP |

### Tratamentos Especiais com UPPER/TRIM

| Valor Original (case-insensitive) | GRE_SIMPLIFICADA |
|-----------------------------------|------------------|
| NÃO ATRIBUÍDO / NAO ATRIBUIDO | Sem distribuição |
| SEM CLASSIFICAÇÃO / SEM CLASSIFICACAO | SEM CLASSIFICAÇÃO |
| (vazio ou NULL) | Sem distribuição |

## Exemplos Práticos no Resultado

Dados reais mostrando simplificações:

| NR_PROT | ATRIBUIDO | DEPARTMENT | GRE | GRE_SIMPLIFICADA |
|---------|-----------|------------|-----|------------------|
| PR0309572023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO | ACERVO |
| PR0310732023 | rosana.torre4031 | EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO | ACERVO | ACERVO |
| PR0311262023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 | GRE10 |
| PR0311962023 | lucas.rodrigues4435 | GRE10 ARARAQUARA | GRE10 | GRE10 |

## Filtros WHERE Aplicados

A query exclui determinadas GREs simplificadas do resultado final:

### Exclusões por Valor Exato (NOT IN)

```sql
WHERE ... NOT IN (
    'SEM CLASSIFICAÇÃO',
    'PRESIDÊNCIA',
    'EISI',
    'GDEP',
    'GFISC',
    'GERENCIA DE DES. E EXECUÇÃO DE PROJETOS - GDEP',
    'GERENCIA DE FISCALIZAÇÃO E PROCEDIMENTOS - GFISC',
    'ADM DADOS',
    'CONSULTA TÉCNICA',
    'EQUIPE OPERAÇÃO',
    'GEJ',
    'JURÍDICO'
)
```

### Exclusões por LIKE (Pattern Matching)

```sql
AND UPPER(...) NOT LIKE '%ADM DE DADOS%'
AND UPPER(...) NOT LIKE '%CONSULTA TÉCNICA%'
AND UPPER(...) NOT LIKE '%CONSULTA TECNICA%'
AND UPPER(...) NOT LIKE '%EQUIPE OPERAÇÃO%'
AND UPPER(...) NOT LIKE '%EQUIPE OPERACAO%'
AND UPPER(...) NOT LIKE '%GEJ%'
AND UPPER(...) NOT LIKE '%GDEP%'
AND UPPER(...) NOT LIKE '%GFISC%'
```

**Importante**: A lógica de exclusão no WHERE repete TODA a lógica de GRE_Simplificada para garantir que os filtros sejam aplicados corretamente.

## Diferenças entre GRE e GRE_SIMPLIFICADA

| Aspecto | GRE | GRE_SIMPLIFICADA |
|---------|-----|------------------|
| **EQUIPE DE ATEND. AOS PROF...** | Mantém nome completo ou "ACERVO" | Sempre "ACERVO" |
| **Aguardando Pagamento** | Mantém status | "Sem distribuição" |
| **Departamentos longos** | Nome completo | Sigla simplificada |
| **Filtros WHERE** | Não aplicados | Excluídos do resultado |

## Hierarquia de Prioridades

```
1º - Cancelado por não pagamento
2º - ACERVO (funcionário com histórico)
3º - Sem distribuição (múltiplas condições)
4º - GREs regionais (GRE1-GRE12)
5º - Não atribuído → Sem distribuição
6º - Vencido sem pagamento → CANCELADO SEM PAGTO
7º - Departamentos com simplificação
8º - Casos especiais UPPER/TRIM
9º - ELSE: mantém valor original
```

## Caso Especial: Correção ATENDIMENTO → ACERVO

```sql
-- IMPORTANTE: Correção aplicada na query
WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT) = 'EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO' 
THEN 'ACERVO'  -- CORRIGIDO: Era 'ATENDIMENTO', agora é 'ACERVO'
```

Esta correção unifica o tratamento de protocolos de atendimento sob a categoria ACERVO.

## WITH UR no Final da Query

A query termina com:

```sql
WITH UR
```

Esta cláusula indica que a query usa o nível de isolamento "Uncommitted Read" (UR), permitindo leitura de dados não confirmados para melhor performance em relatórios.

## Observações Técnicas

- Simplifica nomes longos em siglas padronizadas
- Agrupa variações do mesmo departamento
- Facilita criação de dashboards e relatórios
- Reduz complexidade para usuários finais
- Aplicação de filtros WHERE garante foco em áreas operacionais principais
- Correção importante: ATENDIMENTO foi unificado com ACERVO
- COALESCE(GRE_FROM_MAPPING, DEPARTMENT) tenta primeiro o mapeamento, depois o departamento
- UPPER() e TRIM() tratam variações de digitação
- Total de aproximadamente 15 departamentos excluídos pelos filtros WHERE

---