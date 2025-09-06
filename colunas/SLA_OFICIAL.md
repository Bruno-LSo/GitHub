# SLA_OFICIAL - Prazo Regulamentar

## Descrição Geral

A coluna `SLA_OFICIAL` define o prazo regulamentar em dias úteis para cada tipo de serviço, baseado no subtipo do protocolo conforme normas do CREA-SP.

## Origem dos Dados

### Cálculo Baseado em Regras de Negócio
- **Tabela de Origem**: `DB2INET.TB_SUBTIPO_PROTOCOLO`
- **Coluna Base**: `NM_SUBTP_PROT` (Nome do Subtipo de Protocolo)
- **Tipo de Dado**: INTEGER - Número de dias
- **Definição**: Mapeamento fixo por subtipo com valor padrão de 20 dias

## Regra Completa de Mapeamento

O SLA_OFICIAL é calculado na CTE PROTOCOLO_COMPLETO com todas estas regras:

```sql
-- Na CTE PROTOCOLO_COMPLETO:
CASE
    WHEN STP.NM_SUBTP_PROT = 'Alteração de Registro de Empresa' THEN 20
    WHEN STP.NM_SUBTP_PROT IN ('Baixa Art - BAIXA DO RESP. PERANTE EMPR. CONTRATADA',
                                'Baixa Art - PARALISAÇÃO DA OBRA/SERVIÇO',
                                'Baixa Art - RESCISÃO CONTRATUAL',
                                'Baixa Art - SUBSTITUIÇÃO DO RESPONSÁVEL TÉCNICO') THEN 10
    WHEN STP.NM_SUBTP_PROT = 'CADASTRO DE IES' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'CADASTRO DE RESPONSAVEL IES' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Cancelamento de Registro de Empresa' THEN 20
    WHEN STP.NM_SUBTP_PROT IN ('Cancelar Art - CONTRATO NÃO FOI EXECUTADO',
                                'Cancelar Art - DUPLICIDADE DE PAGAMENTO - DN Nº 85/2011',
                                'Cancelar Art - NENHUMA DAS ATIVIDADES TÉCNICAS FORAM EXECUTADAS') THEN 10
    WHEN STP.NM_SUBTP_PROT IN ('CAT Com Registro de Atestado - Atividade Concluída',
                                'CAT Com Registro de Atestado - Atividade em Andamento',
                                'CAT com Registro de Atestado - Complementar',
                                'CAT de atividade desenvolvida no exterior',
                                'CAT Sem Registro de Atestado') THEN 20
    WHEN STP.NM_SUBTP_PROT = 'DESCONTO DE ANUIDADE' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'ENVIO DE LISTA DE FORMANDOS' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Interrupção de Registro' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Interrupção de Registro de Empresa' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'PREMIAÇÃO DICENTE' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Reativar Registro' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'REEMBOLSO DE TAXA' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Registro com atestado - RU' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Registro com atestado CREANET' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Registro com diploma - RU' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Registro com diploma CREANET' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Registro de Empresa' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'REGISTRO E ATUALIZAÇÃO DE CURSO' THEN 15
    WHEN STP.NM_SUBTP_PROT = 'Segunda Via' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Substituição de CAT com Novo Atestado' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Validação Prévia para Cartório em Instrumento de Constituição, Alteração ou Dissolução' THEN 20
    WHEN STP.NM_SUBTP_PROT = 'Visto de Profissional' THEN 15
    ELSE 20  -- Padrão para subtipos não mapeados
END AS SLA_OFICIAL
```

## Categorização Completa por Prazo

### Serviços com 10 dias de prazo:

```sql
WHEN STP.NM_SUBTP_PROT IN (
    'Baixa Art - BAIXA DO RESP. PERANTE EMPR. CONTRATADA',
    'Baixa Art - PARALISAÇÃO DA OBRA/SERVIÇO',
    'Baixa Art - RESCISÃO CONTRATUAL',
    'Baixa Art - SUBSTITUIÇÃO DO RESPONSÁVEL TÉCNICO'
) THEN 10

WHEN STP.NM_SUBTP_PROT IN (
    'Cancelar Art - CONTRATO NÃO FOI EXECUTADO',
    'Cancelar Art - DUPLICIDADE DE PAGAMENTO - DN Nº 85/2011',
    'Cancelar Art - NENHUMA DAS ATIVIDADES TÉCNICAS FORAM EXECUTADAS'
) THEN 10
```

**Total**: 7 subtipos com prazo de 10 dias

### Serviços com 15 dias de prazo:

```sql
WHEN STP.NM_SUBTP_PROT = 'CADASTRO DE IES' THEN 15
WHEN STP.NM_SUBTP_PROT = 'ENVIO DE LISTA DE FORMANDOS' THEN 15
WHEN STP.NM_SUBTP_PROT = 'Interrupção de Registro' THEN 15
WHEN STP.NM_SUBTP_PROT = 'Reativar Registro' THEN 15
WHEN STP.NM_SUBTP_PROT = 'Registro com atestado CREANET' THEN 15
WHEN STP.NM_SUBTP_PROT = 'Registro com diploma - RU' THEN 15
WHEN STP.NM_SUBTP_PROT = 'Registro com diploma CREANET' THEN 15
WHEN STP.NM_SUBTP_PROT = 'REGISTRO E ATUALIZAÇÃO DE CURSO' THEN 15
WHEN STP.NM_SUBTP_PROT = 'Visto de Profissional' THEN 15
```

**Total**: 9 subtipos com prazo de 15 dias

### Serviços com 20 dias de prazo:

```sql
WHEN STP.NM_SUBTP_PROT = 'Alteração de Registro de Empresa' THEN 20
WHEN STP.NM_SUBTP_PROT = 'CADASTRO DE RESPONSAVEL IES' THEN 20
WHEN STP.NM_SUBTP_PROT = 'Cancelamento de Registro de Empresa' THEN 20

WHEN STP.NM_SUBTP_PROT IN (
    'CAT Com Registro de Atestado - Atividade Concluída',
    'CAT Com Registro de Atestado - Atividade em Andamento',
    'CAT com Registro de Atestado - Complementar',
    'CAT de atividade desenvolvida no exterior',
    'CAT Sem Registro de Atestado'
) THEN 20

WHEN STP.NM_SUBTP_PROT = 'DESCONTO DE ANUIDADE' THEN 20
WHEN STP.NM_SUBTP_PROT = 'Interrupção de Registro de Empresa' THEN 20
WHEN STP.NM_SUBTP_PROT = 'PREMIAÇÃO DICENTE' THEN 20
WHEN STP.NM_SUBTP_PROT = 'REEMBOLSO DE TAXA' THEN 20
WHEN STP.NM_SUBTP_PROT = 'Registro com atestado - RU' THEN 20
WHEN STP.NM_SUBTP_PROT = 'Registro de Empresa' THEN 20
WHEN STP.NM_SUBTP_PROT = 'Segunda Via' THEN 20
WHEN STP.NM_SUBTP_PROT = 'Substituição de CAT com Novo Atestado' THEN 20
WHEN STP.NM_SUBTP_PROT = 'Validação Prévia para Cartório em Instrumento de Constituição, Alteração ou Dissolução' THEN 20

ELSE 20  -- Padrão para subtipos não mapeados
```

**Total**: 14 subtipos explícitos + padrão para não mapeados

## Tabela Resumo de Todos os Subtipos Mapeados

| Subtipo | Prazo (dias) | Categoria |
|---------|--------------|-----------|
| **Baixa Art - BAIXA DO RESP. PERANTE EMPR. CONTRATADA** | 10 | Baixa Art |
| **Baixa Art - PARALISAÇÃO DA OBRA/SERVIÇO** | 10 | Baixa Art |
| **Baixa Art - RESCISÃO CONTRATUAL** | 10 | Baixa Art |
| **Baixa Art - SUBSTITUIÇÃO DO RESPONSÁVEL TÉCNICO** | 10 | Baixa Art |
| **Cancelar Art - CONTRATO NÃO FOI EXECUTADO** | 10 | Cancelamento Art |
| **Cancelar Art - DUPLICIDADE DE PAGAMENTO - DN Nº 85/2011** | 10 | Cancelamento Art |
| **Cancelar Art - NENHUMA DAS ATIVIDADES TÉCNICAS FORAM EXECUTADAS** | 10 | Cancelamento Art |
| **CADASTRO DE IES** | 15 | Instituição de Ensino |
| **ENVIO DE LISTA DE FORMANDOS** | 15 | Instituição de Ensino |
| **Interrupção de Registro** | 15 | Registro Profissional |
| **Reativar Registro** | 15 | Registro Profissional |
| **Registro com atestado CREANET** | 15 | Registro Profissional |
| **Registro com diploma - RU** | 15 | Registro Profissional |
| **Registro com diploma CREANET** | 15 | Registro Profissional |
| **REGISTRO E ATUALIZAÇÃO DE CURSO** | 15 | Instituição de Ensino |
| **Visto de Profissional** | 15 | Registro Profissional |
| **Alteração de Registro de Empresa** | 20 | Registro Empresa |
| **CADASTRO DE RESPONSAVEL IES** | 20 | Instituição de Ensino |
| **Cancelamento de Registro de Empresa** | 20 | Registro Empresa |
| **CAT Com Registro de Atestado - Atividade Concluída** | 20 | Acervo Técnico |
| **CAT Com Registro de Atestado - Atividade em Andamento** | 20 | Acervo Técnico |
| **CAT com Registro de Atestado - Complementar** | 20 | Acervo Técnico |
| **CAT de atividade desenvolvida no exterior** | 20 | Acervo Técnico |
| **CAT Sem Registro de Atestado** | 20 | Acervo Técnico |
| **DESCONTO DE ANUIDADE** | 20 | Financeiro |
| **Interrupção de Registro de Empresa** | 20 | Registro Empresa |
| **PREMIAÇÃO DICENTE** | 20 | Institucional |
| **REEMBOLSO DE TAXA** | 20 | Financeiro |
| **Registro com atestado - RU** | 20 | Registro Profissional |
| **Registro de Empresa** | 20 | Registro Empresa |
| **Segunda Via** | 20 | Documentos |
| **Substituição de CAT com Novo Atestado** | 20 | Acervo Técnico |
| **Validação Prévia para Cartório em Instrumento de Constituição, Alteração ou Dissolução** | 20 | Registro Empresa |

## Exemplos Práticos no Resultado

Dados reais da query com seus respectivos SLAs oficiais:

| NR_PROT | NM_TP_PROT | NM_SUBTP_PROT | SLA_OFICIAL | SLA | CATEGORIA_STATUS |
|---------|------------|---------------|-------------|-----|------------------|
| PR0309572023 | Registro de Profissional | Interrupção de Registro | 15 | 25 | Fora do Prazo |
| PR0310732023 | Registro de Profissional | Reativar Registro | 15 | 27 | Fora do Prazo |
| PR0311262023 | Registro de Profissional | Reativar Registro | 15 | 52 | Sem pagamento |
| PR0311962023 | Registro de Profissional | Reativar Registro | 15 | 52 | Sem pagamento |

## Utilização na CATEGORIA_STATUS

O SLA_OFICIAL é usado para determinar se o protocolo está dentro ou fora do prazo:

```sql
-- Na coluna CATEGORIA_STATUS:
WHEN (CASE
        WHEN DAYS_PAGAMENTO IS NOT NULL AND FINISHED = 1 
            THEN DAYS_STATUS - DAYS_PAGAMENTO
        WHEN DAYS_PAGAMENTO IS NOT NULL 
            THEN DAYS_CURRENT - DAYS_PAGAMENTO
        WHEN FINISHED = 1 
            THEN DAYS_STATUS - DAYS_ABERTURA
        ELSE DAYS_CURRENT - DAYS_ABERTURA
     END) <= SLA_OFICIAL
THEN 'Dentro do Prazo'
ELSE 'Fora do Prazo'
```

## Estrutura da Tabela TB_SUBTIPO_PROTOCOLO

Baseando-se nos dados fornecidos:

```sql
-- Estrutura:
NM_SUBTP_PROT | CAD_DOMINIO_GUID | CD_TP_PROT | IND_ATIVO | CD_IDENT | NM_GRUPO_ANALISE_AD | ST_PRE_DISTRIB

-- Exemplos:
1 | Registro com atestado - RU | 35659cad-8b5b-4584-9ec6-7cc5fb98fa09 | 1 | 1 | | PRF-RU
2 | Registro com diploma - RU | 5293c88d-4606-4def-85dc-60c46936fbb5 | 1 | 1 | | PRF-RU
```

## Casos Especiais e Observações

### Cláusula ELSE - Padrão de 20 dias

```sql
ELSE 20  -- Padrão para subtipos não mapeados
```

**Importante**: Qualquer subtipo não explicitamente mapeado recebe automaticamente 20 dias de prazo. Isso garante que novos subtipos criados no sistema tenham um prazo padrão até serem configurados.

### Comparação Case-Sensitive

Os nomes dos subtipos devem corresponder EXATAMENTE, incluindo:
- Acentuação (ç, ã, é, etc.)
- Maiúsculas/minúsculas
- Espaços
- Caracteres especiais

### Relacionamento com NM_TP_PROT

Embora o SLA_OFICIAL seja baseado apenas no subtipo, existe relação lógica com o tipo:
- Tipos "Baixa Art" e "Cancelamento Art" → geralmente 10 dias
- Tipo "Registro de Profissional" → varia entre 15 e 20 dias
- Tipo "Acervo Técnico" (CATs) → sempre 20 dias
- Tipo "Empresa" → sempre 20 dias

## Observações Técnicas

- Valores fixos definidos por regulamentação do CREA-SP
- Não considera feriados ou dias úteis (usa dias corridos)
- Calculado uma única vez na CTE PROTOCOLO_COMPLETO para otimização
- Usado como referência para determinar conformidade com prazos regulamentares
- Essencial para relatórios de performance e conformidade
- Total de 30 subtipos explicitamente mapeados + regra padrão

---