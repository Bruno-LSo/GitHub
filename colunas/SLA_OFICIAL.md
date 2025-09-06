# SLA_OFICIAL - Prazo Regulamentar

## Descrição Geral

A coluna `SLA_OFICIAL` define o prazo regulamentar em dias úteis para cada tipo de serviço, baseado no subtipo do protocolo conforme normas do CREA-SP.

## Origem dos Dados

### Cálculo Baseado em Regras
- **Base**: `NM_SUBTP_PROT` (Nome do Subtipo)
- **Tipo de Dado**: INTEGER - Número de dias
- **Definição**: Mapeamento fixo por subtipo

## Regra de Mapeamento

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

## Categorização por Prazo

### Serviços com 10 dias:
- Baixa Art (todos os tipos)
- Cancelar Art (todos os tipos)

### Serviços com 15 dias:
- CADASTRO DE IES
- ENVIO DE LISTA DE FORMANDOS
- Interrupção de Registro
- Reativar Registro
- Registro com atestado CREANET
- Registro com diploma (RU e CREANET)
- REGISTRO E ATUALIZAÇÃO DE CURSO
- Visto de Profissional

### Serviços com 20 dias:
- Alteração de Registro de Empresa
- CADASTRO DE RESPONSAVEL IES
- Cancelamento de Registro de Empresa
- CAT (todos os tipos)
- DESCONTO DE ANUIDADE
- Interrupção de Registro de Empresa
- PREMIAÇÃO DICENTE
- REEMBOLSO DE TAXA
- Registro com atestado - RU
- Registro de Empresa
- Segunda Via
- Substituição de CAT com Novo Atestado
- Validação Prévia para Cartório

## Exemplos Práticos no Resultado

Dados reais com SLA_OFICIAL:

| NR_PROT | NM_SUBTP_PROT | SLA_OFICIAL | SLA | Situação |
|---------|---------------|-------------|-----|----------|
| PR0309572023 | Interrupção de Registro | 15 | 25 | Atrasado (25 > 15) |
| PR0310732023 | Reativar Registro | 15 | 27 | Atrasado (27 > 15) |
| PR0311262023 | Reativar Registro | 15 | 52 | Atrasado (52 > 15) |
| PR0311962023 | Reativar Registro | 15 | 52 | Atrasado (52 > 15) |

## Utilização na CATEGORIA_STATUS

O SLA_OFICIAL é usado para determinar se está dentro ou fora do prazo:

```sql
-- Comparação com SLA calculado:
WHEN (tempo_decorrido) <= SLA_OFICIAL
THEN 'Dentro do Prazo'
ELSE 'Fora do Prazo'
```

## Observações Técnicas

- Valores fixos definidos por regulamentação do CREA-SP
- Não considera feriados ou dias úteis (usa dias corridos)
- Padrão de 20 dias para subtipos não mapeados
- Base para cálculo de indicadores de performance
- Essencial para relatórios de conformidade com prazos regulamentares

---
