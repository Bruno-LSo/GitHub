#  DOCUMENTO COMPLETO DE EXIGÊNCIAS - VERSÃO ATUALIZADA

##  **EXIGÊNCIAS JÁ IMPLEMENTADAS NA QUERY ATUAL**

### 1. **EXIGÊNCIA ATENDIDA = ANÁLISE**
**Fonte no documento:**
```
"exigência atendida = Análise"
```
**Status:**  **JÁ IMPLEMENTADO**

**Onde está na query atual:**
```sql
-- Linhas 147-159 da query, coluna DS_STATUS_Simplificada:
CASE
    ...
    WHEN PCG.DS_STATUS = 'Exigência Atendida' THEN 'Aguardando Análise'  -- AQUI!
    ...
END AS DS_STATUS_SIMPLIFICADA
```

---

### 2. **STATUS CLASSIFICAÇÃO ABERTO/FECHADO/EXIGÊNCIA**
**Fonte no documento:**
```
"Status SLA - Aberto e Fechado"
"'cancelado', 'deferido' e 'indeferido' serão agora classificados como 'FECHADO'..."
```
**Status:**  **JÁ IMPLEMENTADO**

**Onde está na query atual:**
```sql
-- Linhas 162-175 da query, coluna DS_STATUS_Classificacao:
CASE
    WHEN PCG.DS_STATUS = 'Solicitação Deferida' THEN 'Fechado'
    WHEN PCG.DS_STATUS = 'Solicitação Cancelada' THEN 'Fechado'
    WHEN PCG.DS_STATUS = 'Solicitação Indeferida' THEN 'Fechado'
    WHEN PCG.DS_STATUS = 'Exigência Atendida' THEN 'Exigência'
    WHEN PCG.DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
    ELSE 'Aberto'
END AS DS_STATUS_Classificacao
```

---

##  **EXIGÊNCIAS QUE PRECISAM SER IMPLEMENTADAS**

### 3. **NOVA ESTRUTURA DE SLA DUPLO E CATEGORIA_STATUS ATUALIZADA**
**Fonte no documento:**
```
"SLA OFICIAL - Tempo oficial... SLA PROTOCOLO - Tempo que o protocolo levou..."
[Tabela com 34 subtipos e prazos]
"Por exemplo alteração de registro de empresa maior que 20 dias de SLA, 
Fora do Prazo, se estiver menor ou igual dentro do prazo"
```

**O que modificar:**

**A) Adicionar coluna SLA_OFICIAL após SLA (linha 207):**
```sql
-- Nova coluna SLA_OFICIAL
CASE
    WHEN PCG.NM_SUBTP_PROT = 'Alteração de Registro de Empresa' THEN 20
    WHEN PCG.NM_SUBTP_PROT IN ('Baixa Art - BAIXA DO RESP. PERANTE EMPR. CONTRATADA',
                                'Baixa Art - PARALISAÇÃO DA OBRA/SERVIÇO',
                                'Baixa Art - RESCISÃO CONTRATUAL',
                                'Baixa Art - SUBSTITUIÇÃO DO RESPONSÁVEL TÉCNICO') THEN 10
    WHEN PCG.NM_SUBTP_PROT = 'CADASTRO DE IES' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'CADASTRO DE RESPONSAVEL IES' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'Cancelamento de Registro de Empresa' THEN 20
    WHEN PCG.NM_SUBTP_PROT IN ('Cancelar Art - CONTRATO NÃO FOI EXECUTADO',
                                'Cancelar Art - DUPLICIDADE DE PAGAMENTO - DN Nº 85/2011',
                                'Cancelar Art - NENHUMA DAS ATIVIDADES TÉCNICAS FORAM EXECUTADAS') THEN 10
    WHEN PCG.NM_SUBTP_PROT IN ('CAT Com Registro de Atestado - Atividade Concluída',
                                'CAT Com Registro de Atestado - Atividade em Andamento',
                                'CAT com Registro de Atestado - Complementar',
                                'CAT de atividade desenvolvida no exterior',
                                'CAT Sem Registro de Atestado') THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'DESCONTO DE ANUIDADE' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'ENVIO DE LISTA DE FORMANDOS' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'Interrupção de Registro' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'Interrupção de Registro de Empresa' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'PREMIAÇÃO DICENTE' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'Reativar Registro' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'REEMBOLSO DE TAXA' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'Registro com atestado - RU' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'Registro com atestado CREANET' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'Registro com diploma - RU' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'Registro com diploma CREANET' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'Registro de Empresa' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'REGISTRO E ATUALIZAÇÃO DE CURSO' THEN 15
    WHEN PCG.NM_SUBTP_PROT = 'Segunda Via' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'Substituição de CAT com Novo Atestado' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'Validação Prévia para Cartório em Instrumento de Constituição, Alteração ou Dissolução' THEN 20
    WHEN PCG.NM_SUBTP_PROT = 'Visto de Profissional' THEN 15
    ELSE 20  -- Padrão para não mapeados
END AS SLA_OFICIAL,
```

**B) Substituir CATEGORIA_STATUS (linhas 209-234) por:**
```sql
CASE
    -- Manter casos especiais
    WHEN CURRENT DATE > PCG.DT_EST_CONCLUSAO AND PCG.DT_PAGAMENTO IS NULL AND PCG.NR_BOLETO IS NOT NULL
        THEN 'Sem pagamento'
    
    -- Nova lógica baseada em SLA_OFICIAL
    WHEN (CASE
            WHEN PCG.DT_PAGAMENTO IS NOT NULL AND PCG.FINISHED = 1 
                THEN PCG.DAYS_STATUS - PCG.DAYS_PAGAMENTO
            WHEN PCG.DT_PAGAMENTO IS NOT NULL 
                THEN PCG.DAYS_CURRENT - PCG.DAYS_PAGAMENTO
            WHEN PCG.FINISHED = 1 
                THEN PCG.DAYS_STATUS - PCG.DAYS_ABERTURA
            ELSE PCG.DAYS_CURRENT - PCG.DAYS_ABERTURA
         END) <= 
         (CASE -- SLA_OFICIAL inline
            WHEN PCG.NM_SUBTP_PROT = 'Alteração de Registro de Empresa' THEN 20
            -- [repetir todos os 34 mapeamentos]
            ELSE 20
         END)
    THEN 'Dentro do Prazo'
    ELSE 'Fora do Prazo'
END AS CATEGORIA_STATUS,
```

---

### 4. **REGRA DO ACERVO**
**Fonte no documento:**
```
"para os casos de Acervo, se o funcionário tiver algum protocolo de Acervo, 
considerar na GRE como ACERVO"
```

**O que modificar:**

**A) Adicionar CTE após PROTOCOLO_BASE (linha 32):**
```sql
FUNCIONARIOS_ACERVO AS (
    SELECT DISTINCT 
        UPPER(SP.ATRIBUIDO) AS ATRIBUIDO_UPPER
    FROM DB2INET.TB_PROTOCOLO_CREANET P
    INNER JOIN DB2INET.TB_STATUS_PROTOCOLO_CREANET SP ON SP.CD_PROT = P.CD_PROT
    INNER JOIN DB2INET.TB_TIPO_PROTOCOLO TP ON P.CD_TP_PROT = TP.CD_TP_PROT
    WHERE TP.NM_TP_PROT = 'Acervo Técnico'
    AND SP.ATRIBUIDO IS NOT NULL
    AND TRIM(SP.ATRIBUIDO) <> ''
    AND P.DRAFT = 0
),
```

**B) Modificar PROTOCOLO_COM_GRE (linha 75), adicionar no início do CASE:**
```sql
CASE
    WHEN EXISTS (
        SELECT 1 FROM FUNCIONARIOS_ACERVO FA 
        WHERE FA.ATRIBUIDO_UPPER = UPPER(PCC.ATRIBUIDO)
    ) THEN 'ACERVO'
    -- Resto continua...
```

**C) Modificar GRE_Simplificada (linha 245), adicionar:**
```sql
CASE
    WHEN PCG.GRE_CALCULADA = 'ACERVO' THEN 'ACERVO'
    -- Resto continua...
```

---

### 5. **IDENTIFICAÇÃO "CANCELADO POR NÃO PAGAMENTO"**
**Fonte no documento:**
```
"se a solicitação esta cancelada, com status fechada e na observação 
estiver Boleto Vencido"
```

**O que modificar:**

**A) Em GRE_CALCULADA (linha 75), adicionar como primeira condição:**
```sql
CASE
    WHEN PCC.DS_STATUS = 'Solicitação Cancelada' 
         AND PCC.FINISHED = 1 
         AND PCC.DS_OBSERVACOES LIKE '%Boleto Vencido%'
    THEN 'Cancelado por não pagamento'
    -- Resto continua...
```

**B) Em SLA (linha 185), adicionar como primeira condição:**
```sql
CASE
    WHEN PCG.DS_STATUS = 'Solicitação Cancelada' 
         AND PCG.FINISHED = 1 
         AND PCG.DS_OBSERVACOES LIKE '%Boleto Vencido%'
    THEN 0
    -- Resto continua...
```

**C) Em GRE_Simplificada (linha 245), adicionar:**
```sql
CASE
    WHEN PCG.GRE_CALCULADA = 'Cancelado por não pagamento' THEN 'Cancelado por não pagamento'
    -- Resto continua...
```

**D) Em CATEGORIA_STATUS, adicionar como primeira condição:**
```sql
CASE
    WHEN PCG.DS_STATUS = 'Solicitação Cancelada' 
         AND PCG.FINISHED = 1 
         AND PCG.DS_OBSERVACOES LIKE '%Boleto Vencido%'
    THEN 'Cancelado por não pagamento'
    -- Resto da lógica de prazo...
```

---

### 6. **TRATAMENTO LOGIN = 0**
**Fonte no documento:**
```
"o Login esta como 0, então considerar como não atribuido"
```

**O que modificar em PROTOCOLO_COM_GRE (linha 83):**
```sql
WHEN PCC.DS_OBSERVACOES IS NULL OR TRIM(PCC.DS_OBSERVACOES) = '' 
     OR PCC.ATRIBUIDO IS NULL OR TRIM(PCC.ATRIBUIDO) = ''
     OR PCC.ATRIBUIDO = '0'  -- ADICIONAR ESTA CONDIÇÃO
THEN
    CASE
        WHEN CURRENT DATE > PCC.DT_EST_CONCLUSAO AND PCC.DT_PAGAMENTO IS NULL 
             AND PCC.NR_BOLETO IS NOT NULL
            THEN 'Sem pagamento'
        ELSE 'Protocolo ainda não atribuído a um agente'
    END
```

---

### 7. **AJUSTES DE TEXTO EM GRE_SIMPLIFICADA**
**Fonte no documento:**
```
"sem pagto muda para: cancelado sem pagto"
"aguardando pagamento vai aparecer como: sem distribuição"
```

**O que modificar em GRE_Simplificada (linha 245-280):**
```sql
CASE
    ...
    -- ADICIONAR condição para Aguardando Pagamento
    WHEN PCG.DS_STATUS = 'Aguardando Pagamento' THEN 'Sem distribuição'
    
    -- MODIFICAR texto
    WHEN PCG.GRE_CALCULADA = 'Sem pagamento' THEN 'CANCELADO SEM PAGTO'  -- Era 'SEM PAGTO'
    ...
```

---

### 8. **EXCLUSÃO DE REGISTROS PRESIDÊNCIA**
**Fonte no documento:**
```
"presidência vai sair - serão removidas as linhas"
```

**O que modificar no WHERE (após linha 391), ADICIONAR:**
```sql
AND CASE
    -- Repetir todo o CASE da GRE_Simplificada
    ...
END NOT IN ('SEM CLASSIFICAÇÃO', 'PRESIDÊNCIA')  -- Adicionar PRESIDÊNCIA
```

---

### 9. **DEPARTAMENTOS TESTE (DOCUMENTAÇÃO)**
**Fonte no documento:**
```
"Os resultados a seguir serão considerados TESTES"
```
**Status:** 📝 **APENAS DOCUMENTAÇÃO** - Não requer mudança na query

---

##  **CHECKLIST FINAL**

| # | Exigência | Status | Impacto |
|---|-----------|--------|---------|
| 1 | Exigência Atendida = Análise | ✅ Implementado | Nenhum |
| 2 | DS_STATUS_Classificacao | ✅ Implementado | Nenhum |
| 3 | SLA Duplo + CATEGORIA_STATUS | ❌ Pendente | Nova coluna + Refazer lógica |
| 4 | Regra do Acervo | ❌ Pendente | Nova CTE + Lógicas |
| 5 | Cancelado por não pagamento | ❌ Pendente | Múltiplas condições |
| 6 | Login = 0 | ❌ Pendente | Uma condição |
| 7 | Ajustes texto GRE | ❌ Pendente | Dois ajustes |
| 8 | Excluir Presidência | ❌ Pendente | Filtro WHERE |
| 9 | Departamentos TESTE | 📝 Documentação | Nenhum |

**Total: 6 implementações necessárias na query**

Query no estado atual antes das modificações:
```sql
WITH PROTOCOLO_BASE AS (
    -- Pre-calcula valores que serão reutilizados
    SELECT
        P.NR_PROT,
        P.DT_ABERTURA,
        P.DT_EST_CONCLUSAO,
        P.CD_TP_PROT,
        P.CD_SUBTP_PROT,
        P.FINISHED,
        P.CD_PROT,
        SP.DS_OBSERVACOES,
        SP.DT_STATUS,
        SP.ATRIBUIDO,
        SP.CD_TP_STATUS_PROT,
        -- Pre-calcula formatações de data
        TO_CHAR(P.DT_ABERTURA,'DD/MM/YYYY') AS DATA_ABERTURA_FMT,
        TO_CHAR(P.DT_EST_CONCLUSAO,'DD/MM/YYYY') AS DATA_EST_CONCLUSAO_FMT,
        TO_CHAR(SP.DT_STATUS,'DD/MM/YYYY') AS DT_STATUS_FMT,
        -- Pre-calcula dias para SLA
        DAYS(P.DT_ABERTURA) AS DAYS_ABERTURA,
        DAYS(SP.DT_STATUS) AS DAYS_STATUS,
        DAYS(CURRENT DATE) AS DAYS_CURRENT,
        -- Pre-calcula diferenças de dias para CATEGORIA_STATUS
        (DAYS(CURRENT DATE) - DAYS(P.DT_ABERTURA)) AS DIAS_DESDE_ABERTURA
    FROM DB2INET.TB_PROTOCOLO_CREANET P
        INNER JOIN DB2INET.TB_STATUS_PROTOCOLO_CREANET SP ON SP.CD_PROT = P.CD_PROT
    WHERE SP.CD_STAT_PROT = (
        SELECT MAX(CD_STAT_PROT)
        FROM DB2INET.TB_STATUS_PROTOCOLO_CREANET
        WHERE CD_PROT = P.CD_PROT
    )
    AND P.DRAFT = 0
),
PROTOCOLO_COM_REFS AS (
    -- Adiciona referências e JOIN com outras tabelas
    SELECT
        PB.*,
        TP_PROT.NM_TP_PROT,
        STP.NM_SUBTP_PROT,
        TSP.DS_STATUS,
        F.DEPARTMENT,
        F.SAMACCOUNTNAME,
        GRES.GRE_DESC,
        BOLETO.NR_BOLETO,
        BOLETO.DT_PAGAMENTO,
        TO_CHAR(BOLETO.NR_BOLETO) AS NR_BOLETO_FMT,
        TO_CHAR(BOLETO.DT_PAGAMENTO,'DD/MM/YYYY') AS DT_PAGAMENTO_FMT,
        CASE WHEN BOLETO.DT_PAGAMENTO IS NOT NULL THEN DAYS(BOLETO.DT_PAGAMENTO) ELSE NULL END AS DAYS_PAGAMENTO
    FROM PROTOCOLO_BASE PB
        INNER JOIN DB2INET.TB_TIPO_PROTOCOLO TP_PROT ON PB.CD_TP_PROT = TP_PROT.CD_TP_PROT
        INNER JOIN DB2INET.TB_SUBTIPO_PROTOCOLO STP ON PB.CD_SUBTP_PROT = STP.CD_SUBTP_PROT
        INNER JOIN DB2INET.TB_TIPO_STATUS_PROTOCOLO TSP ON TSP.CD_TP_STATUS_PROT = PB.CD_TP_STATUS_PROT
        LEFT JOIN DB2INET.TB_FUNCIONARIO F ON UPPER(F.SAMACCOUNTNAME) = UPPER(PB.ATRIBUIDO) AND F.DELETED = 0
        LEFT JOIN DB2INET.TB_BOLETO BOLETO ON BOLETO.CD_PROC = PB.NR_PROT
        LEFT JOIN DB2INET.VW_UNIDADES_GRES GRES ON GRES.DESC_UNI = F.DEPARTMENT
),
PROTOCOLO_COM_CIDADE AS (
    -- Adiciona CIDADE através de subconsulta otimizada (mantém FETCH FIRST 1 ROW ONLY)
    SELECT
        PCR.*,
        (SELECT L.LOC_NO
         FROM DB2INET.TB_UNIDADE U2
         INNER JOIN DB2INET.TB_UNIDADE_LOCALIDADE UL2 ON UL2.CD_UNI = U2.CD_UNI
         INNER JOIN DB2INET.TB_LOCALIDADE L ON L.LOC_NU = UL2.LOC_NU
         WHERE (U2.DESC_UNI = PCR.DEPARTMENT OR U2.DESC_SIGLA = PCR.DEPARTMENT)
         FETCH FIRST 1 ROW ONLY) AS CIDADE
    FROM PROTOCOLO_COM_REFS PCR
),
PROTOCOLO_COM_GRE AS (
    -- Calcula GRE uma única vez
    SELECT
        PCC.*,
        CASE
            -- NOVA REGRA: "Sem distribuição" quando tem boleto pago mas sem distribuição
            WHEN PCC.NR_BOLETO IS NOT NULL AND PCC.DT_PAGAMENTO IS NOT NULL
                AND (PCC.DEPARTMENT IS NULL OR TRIM(PCC.DEPARTMENT) = '')
                AND (PCC.GRE_DESC IS NULL OR TRIM(PCC.GRE_DESC) = '')  
                AND (PCC.CIDADE IS NULL OR TRIM(PCC.CIDADE) = '')
                AND (PCC.ATRIBUIDO IS NULL OR TRIM(PCC.ATRIBUIDO) = '')
                THEN 'Sem distribuição'
            WHEN PCC.DS_OBSERVACOES IS NULL OR TRIM(PCC.DS_OBSERVACOES) = '' OR PCC.ATRIBUIDO IS NULL OR TRIM(PCC.ATRIBUIDO) = ''
                THEN
                    CASE
                        WHEN CURRENT DATE > PCC.DT_EST_CONCLUSAO AND PCC.DT_PAGAMENTO IS NULL AND PCC.NR_BOLETO IS NOT NULL
                            THEN 'Sem pagamento'
                        ELSE 'Protocolo ainda não atribuído a um agente'
                    END
            ELSE
                COALESCE(
                    (SELECT VW_GRE.GRE_SIGLA
                     FROM DB2INET.TB_FUNCIONARIO F_GRE
                     INNER JOIN DB2INET.TB_UNIDADE U_GRE ON (U_GRE.DESC_UNI = F_GRE.DEPARTMENT OR U_GRE.DESC_SIGLA = F_GRE.DEPARTMENT)
                     INNER JOIN DB2INET.VW_UNIDADES_GRES VW_GRE ON U_GRE.DESC_SIGLA = VW_GRE.SIGLA_UNI
                     WHERE UPPER(F_GRE.SAMACCOUNTNAME) = UPPER(PCC.ATRIBUIDO) AND (F_GRE.DELETED IS NULL OR F_GRE.DELETED = 0)
                     FETCH FIRST 1 ROW ONLY),
                    PCC.DEPARTMENT
                )
        END AS GRE_CALCULADA
    FROM PROTOCOLO_COM_CIDADE PCC
)
 
-- SELECT final otimizado
SELECT
    PCG.NR_PROT,
    PCG.DATA_ABERTURA_FMT AS DATA_ABERTURA,
    PCG.DATA_EST_CONCLUSAO_FMT AS DATA_EST_CONCLUSAO,
    PCG.NM_TP_PROT,
    PCG.NM_SUBTP_PROT,
    -- DS_STATUS com regras originais + NOVA REGRA para Solicitação Cancelada
    CASE
        -- NOVA REGRA: Quando tem pagamento, está finalizado e DT_STATUS < DT_PAGAMENTO
        WHEN PCG.DT_PAGAMENTO IS NOT NULL AND PCG.FINISHED = 1 AND PCG.DAYS_STATUS < PCG.DAYS_PAGAMENTO
            THEN 'Solicitação Cancelada'
        WHEN PCG.NR_BOLETO IS NOT NULL AND PCG.DT_PAGAMENTO IS NULL AND PCG.DS_STATUS = 'Solicitação Enviada'
            THEN 'Aguardando Pagamento'
        WHEN PCG.NR_BOLETO IS NULL AND PCG.FINISHED = 0 AND PCG.DS_STATUS = 'Solicitação Enviada'
            THEN 'Aguardando Análise'
        ELSE PCG.DS_STATUS
    END AS DS_STATUS,
    -- DS_STATUS_Simplificada baseada no status ORIGINAL
    CASE
        WHEN PCG.DS_STATUS = 'Solicitação Deferida' THEN 'Solicitação Deferida'
        WHEN PCG.DS_STATUS = 'Solicitação Cancelada' THEN 'Solicitação Cancelada'
        WHEN PCG.DS_STATUS = 'Solicitação Indeferida' THEN 'Solicitação Indeferida'
        WHEN PCG.DS_STATUS = 'Aguardando Análise' THEN 'Aguardando Análise'
        WHEN PCG.DS_STATUS = 'Aguardando doctos. da IE' THEN 'Aguardando doctos. da IE'
        WHEN PCG.DS_STATUS = 'Aguardando Pagamento' THEN 'Aguardando Pagamento'
        WHEN PCG.DS_STATUS = 'Solicitação enviada à câmara especializada' THEN 'Solicitação enviada à câmara especializada'
        WHEN PCG.DS_STATUS = 'Exigência Atendida' THEN 'Aguardando Análise'
        WHEN PCG.DS_STATUS = 'Análise Aprovada' THEN 'Aguardando Análise'
        WHEN PCG.DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
        WHEN PCG.DS_STATUS = 'Solicitação Aguardando Retorno Banco' THEN 'Aguardando Pagamento'
        WHEN PCG.DS_STATUS = 'Solicitação Enviada' THEN 'Aguardando Pagamento'
        ELSE PCG.DS_STATUS
    END AS DS_STATUS_Simplificada,
    -- DS_STATUS_Classificacao baseada no status ORIGINAL
    CASE
        WHEN PCG.DS_STATUS = 'Solicitação Deferida' THEN 'Fechado'
        WHEN PCG.DS_STATUS = 'Solicitação Cancelada' THEN 'Fechado'
        WHEN PCG.DS_STATUS = 'Solicitação Indeferida' THEN 'Fechado'
        WHEN PCG.DS_STATUS = 'Exigência Atendida' THEN 'Exigência'
        WHEN PCG.DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
        WHEN PCG.DS_STATUS = 'Aguardando Análise' THEN 'Aberto'
        WHEN PCG.DS_STATUS = 'Análise Aprovada' THEN 'Aberto'
        WHEN PCG.DS_STATUS = 'Aguardando doctos. da IE' THEN 'Aberto'
        WHEN PCG.DS_STATUS = 'Aguardando Pagamento' THEN 'Aberto'
        WHEN PCG.DS_STATUS = 'Solicitação enviada à câmara especializada' THEN 'Aberto'
        WHEN PCG.DS_STATUS = 'Solicitação Aguardando Retorno Banco' THEN 'Aberto'
        WHEN PCG.DS_STATUS = 'Solicitação Enviada' THEN 'Aberto'
        ELSE 'Aberto'
    END AS DS_STATUS_Classificacao,
    PCG.DS_OBSERVACOES,
    PCG.DT_STATUS_FMT AS DT_STATUS,
    PCG.ATRIBUIDO,
    CASE WHEN PCG.FINISHED = 1 THEN 'SIM' ELSE 'NÃO' END AS FINALIZADO,
    PCG.DEPARTMENT AS UNIDADE,
    PCG.GRE_DESC AS REGIONAL,
    PCG.CIDADE,
    -- SLA otimizado usando dias pré-calculados + NOVA REGRA
    CASE
        -- NOVA REGRA: Quando tem pagamento, está finalizado e DT_STATUS < DT_PAGAMENTO, então SLA = 0
        WHEN PCG.DT_PAGAMENTO IS NOT NULL AND PCG.FINISHED = 1 AND PCG.DAYS_STATUS < PCG.DAYS_PAGAMENTO THEN 0
        -- NOVA REGRA: Quando NM_TP_PROT = 'Empresa' E NM_SUBTP_PROT = 'Registro de Empresa' E DS_STATUS = 'Solicitação Deferida', então SLA = 0
        WHEN PCG.NM_TP_PROT = 'Empresa' AND PCG.NM_SUBTP_PROT = 'Registro de Empresa' AND PCG.DS_STATUS = 'Solicitação Deferida' THEN 0
        WHEN PCG.DAYS_PAGAMENTO IS NOT NULL THEN
            CASE
                WHEN PCG.FINISHED = 1 THEN PCG.DAYS_STATUS - PCG.DAYS_PAGAMENTO
                ELSE PCG.DAYS_CURRENT - PCG.DAYS_PAGAMENTO
            END
        ELSE
            CASE
                WHEN PCG.FINISHED = 1 THEN PCG.DAYS_STATUS - PCG.DAYS_ABERTURA
                ELSE PCG.DAYS_CURRENT - PCG.DAYS_ABERTURA
            END
    END AS SLA,
    -- CATEGORIA_STATUS otimizada usando dias pré-calculados
    CASE
        WHEN CURRENT DATE > PCG.DT_EST_CONCLUSAO AND PCG.DT_PAGAMENTO IS NULL AND PCG.NR_BOLETO IS NOT NULL
            THEN 'Sem pagamento'
        WHEN PCG.CD_TP_PROT IN (3, 10, 1) THEN
            CASE
                WHEN PCG.DIAS_DESDE_ABERTURA <= 8 THEN 'Antes do Prazo'
                WHEN PCG.DIAS_DESDE_ABERTURA BETWEEN 9 AND 19 THEN 'Dentro do Prazo'
                WHEN PCG.DIAS_DESDE_ABERTURA BETWEEN 20 AND 21 THEN 'Em Atraso'
                WHEN PCG.DIAS_DESDE_ABERTURA > 21 THEN 'Crítico'
            END
        WHEN PCG.CD_TP_PROT = 9 THEN
            CASE
                WHEN PCG.DIAS_DESDE_ABERTURA <= 1 THEN 'Antes do Prazo'
                WHEN PCG.DIAS_DESDE_ABERTURA BETWEEN 2 AND 4 THEN 'Dentro do Prazo'
                WHEN PCG.DIAS_DESDE_ABERTURA BETWEEN 5 AND 6 THEN 'Em Atraso'
                WHEN PCG.DIAS_DESDE_ABERTURA > 6 THEN 'Crítico'
            END
        WHEN PCG.CD_TP_PROT IN (12, 2, 4) THEN
            CASE
                WHEN PCG.DIAS_DESDE_ABERTURA <= 7 THEN 'Antes do Prazo'
                WHEN PCG.DIAS_DESDE_ABERTURA BETWEEN 8 AND 12 THEN 'Dentro do Prazo'
                WHEN PCG.DIAS_DESDE_ABERTURA BETWEEN 13 AND 19 THEN 'Em Atraso'
                WHEN PCG.DIAS_DESDE_ABERTURA > 19 THEN 'Crítico'
            END
        ELSE 'Status Não Definido'
    END AS CATEGORIA_STATUS,
    PCG.NR_BOLETO_FMT AS NR_BOLETO,
    PCG.DT_PAGAMENTO_FMT AS DT_PAGAMENTO,
    PCG.GRE_CALCULADA AS GRE,
    -- NOVA COLUNA: DT_FINALIZACAO baseada em DS_STATUS_Simplificada
    CASE
        WHEN (CASE
            WHEN PCG.DS_STATUS = 'Solicitação Deferida' THEN 'Solicitação Deferida'
            WHEN PCG.DS_STATUS = 'Solicitação Cancelada' THEN 'Solicitação Cancelada'
            WHEN PCG.DS_STATUS = 'Solicitação Indeferida' THEN 'Solicitação Indeferida'
            WHEN PCG.DS_STATUS = 'Aguardando Análise' THEN 'Aguardando Análise'
            WHEN PCG.DS_STATUS = 'Aguardando doctos. da IE' THEN 'Aguardando doctos. da IE'
            WHEN PCG.DS_STATUS = 'Aguardando Pagamento' THEN 'Aguardando Pagamento'
            WHEN PCG.DS_STATUS = 'Solicitação enviada à câmara especializada' THEN 'Solicitação enviada à câmara especializada'
            WHEN PCG.DS_STATUS = 'Exigência Atendida' THEN 'Aguardando Análise'
            WHEN PCG.DS_STATUS = 'Análise Aprovada' THEN 'Aguardando Análise'
            WHEN PCG.DS_STATUS = 'Documentação Pendente - Ver Exigências/Observações' THEN 'Exigência'
            WHEN PCG.DS_STATUS = 'Solicitação Aguardando Retorno Banco' THEN 'Aguardando Pagamento'
            WHEN PCG.DS_STATUS = 'Solicitação Enviada' THEN 'Aguardando Pagamento'
            ELSE PCG.DS_STATUS
        END) IN ('Solicitação Deferida', 'Solicitação Cancelada', 'Solicitação Indeferida')
        THEN PCG.DT_STATUS_FMT
        ELSE NULL
    END AS DT_FINALIZACAO,
    -- GRE_Simplificada otimizada usando valor pré-calculado + ALTERAÇÕES SOLICITADAS
    CASE
        -- NOVA REGRA: Mantém "Sem distribuição" quando aplicável
        WHEN PCG.GRE_CALCULADA = 'Sem distribuição' THEN 'Sem distribuição'
        WHEN PCG.GRE_CALCULADA = 'GRE5' THEN 'GRE5'
        WHEN PCG.GRE_CALCULADA = 'GRE12' THEN 'GRE12'
        WHEN PCG.GRE_CALCULADA = 'GRE2' THEN 'GRE2'
        WHEN PCG.GRE_CALCULADA = 'GRE7' THEN 'GRE7'
        WHEN PCG.GRE_CALCULADA = 'GRE9' THEN 'GRE9'
        WHEN PCG.GRE_CALCULADA = 'GRE10' THEN 'GRE10'
        WHEN PCG.GRE_CALCULADA = 'GRE11' THEN 'GRE11'
        WHEN PCG.GRE_CALCULADA = 'GRE3' THEN 'GRE3'
        WHEN PCG.GRE_CALCULADA = 'GRE4' THEN 'GRE4'
        WHEN PCG.GRE_CALCULADA = 'GRE1' THEN 'GRE1'
        WHEN PCG.GRE_CALCULADA = 'GRE8' THEN 'GRE8'
        WHEN PCG.GRE_CALCULADA = 'GRE6' THEN 'GRE6'
        -- ALTERAÇÃO: "Protocolo ainda não atribuído a um agente" agora vira "Sem distribuição"
        WHEN PCG.GRE_CALCULADA = 'Protocolo ainda não atribuído a um agente' THEN 'Sem distribuição'
        WHEN PCG.GRE_CALCULADA = 'Sem pagamento' THEN 'SEM PAGTO'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO' THEN 'ATENDIMENTO'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS' THEN 'ADM DADOS'
        WHEN PCG.GRE_CALCULADA = 'UPS SANTOS' THEN 'UPS SANTOS'
        WHEN PCG.GRE_CALCULADA = 'UNIDADE CAMARAS ESPECIALIZADAS E CONSULTAS TECNICA' THEN 'CONSULTA TÉCNICA'
        WHEN PCG.GRE_CALCULADA = 'GABINETE DA PRESIDENCIA' THEN 'PRESIDÊNCIA'
        WHEN PCG.GRE_CALCULADA = 'UNIDADE FINANCEIRA E CONTABIL - UFC' THEN 'UFC'
        WHEN PCG.GRE_CALCULADA = 'UNIDADE DE PARCERIA E RELACOES INSTITUCIONAIS - UR' THEN 'UR'
        -- ALTERAÇÃO SOLICITADA: Adiciona a versão sem acento
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE SERV ACERVO TECNICO E OPERACIONAL, PROTO' THEN 'ACERVO TÉCNICO'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE SERV ACERVO TÉCNICO E OPERACIONAL, PROTO' THEN 'ACERVO TÉCNICO'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE EXPERIENCIA E JORNADA - GEJ' THEN 'GEJ'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC' THEN 'GFISC'
        WHEN PCG.GRE_CALCULADA = 'UGI REGISTRO' THEN 'UGI REGISTRO'
        WHEN PCG.GRE_CALCULADA = 'EISI' THEN 'EISI'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE OPERACAO, QUALIDADE E SUCESSO DO CLIENTE' THEN 'EQUIPE OPERAÇÃO'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE ASSUNTOS JURIDICOS' THEN 'JURÍDICO'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE DES. E EXECUCAO DE PROJETOS - GDEP' THEN 'GDEP'
        -- NOVAS CONDIÇÕES PARA GARANTIR AS REGRAS:
        -- Se vier "NÃO ATRIBUÍDO" de algum lugar, converte para "Sem distribuição"
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'NÃO ATRIBUÍDO' THEN 'Sem distribuição'
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'NAO ATRIBUIDO' THEN 'Sem distribuição'
        -- Se vier "SEM CLASSIFICAÇÃO", mantém para poder filtrar no WHERE
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'SEM CLASSIFICAÇÃO' THEN 'SEM CLASSIFICAÇÃO'
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'SEM CLASSIFICACAO' THEN 'SEM CLASSIFICAÇÃO'
        -- ALTERAÇÃO: NULL ou vazio agora vira "Sem distribuição"
        WHEN PCG.GRE_CALCULADA IS NULL OR TRIM(PCG.GRE_CALCULADA) = '' THEN 'Sem distribuição'
        ELSE PCG.GRE_CALCULADA
    END AS GRE_Simplificada
 
FROM PROTOCOLO_COM_GRE PCG

-- NOVA CONDIÇÃO WHERE: Exclui linhas onde GRE_Simplificada = 'SEM CLASSIFICAÇÃO'
WHERE CASE
        WHEN PCG.GRE_CALCULADA = 'Sem distribuição' THEN 'Sem distribuição'
        WHEN PCG.GRE_CALCULADA = 'GRE5' THEN 'GRE5'
        WHEN PCG.GRE_CALCULADA = 'GRE12' THEN 'GRE12'
        WHEN PCG.GRE_CALCULADA = 'GRE2' THEN 'GRE2'
        WHEN PCG.GRE_CALCULADA = 'GRE7' THEN 'GRE7'
        WHEN PCG.GRE_CALCULADA = 'GRE9' THEN 'GRE9'
        WHEN PCG.GRE_CALCULADA = 'GRE10' THEN 'GRE10'
        WHEN PCG.GRE_CALCULADA = 'GRE11' THEN 'GRE11'
        WHEN PCG.GRE_CALCULADA = 'GRE3' THEN 'GRE3'
        WHEN PCG.GRE_CALCULADA = 'GRE4' THEN 'GRE4'
        WHEN PCG.GRE_CALCULADA = 'GRE1' THEN 'GRE1'
        WHEN PCG.GRE_CALCULADA = 'GRE8' THEN 'GRE8'
        WHEN PCG.GRE_CALCULADA = 'GRE6' THEN 'GRE6'
        WHEN PCG.GRE_CALCULADA = 'Protocolo ainda não atribuído a um agente' THEN 'Sem distribuição'
        WHEN PCG.GRE_CALCULADA = 'Sem pagamento' THEN 'SEM PAGTO'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE ATEND. AOS PROF, EMP. E INST DE ENSINO' THEN 'ATENDIMENTO'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE DESENVOLVIMENTO E ADMINISTRACAO DE DADOS' THEN 'ADM DADOS'
        WHEN PCG.GRE_CALCULADA = 'UPS SANTOS' THEN 'UPS SANTOS'
        WHEN PCG.GRE_CALCULADA = 'UNIDADE CAMARAS ESPECIALIZADAS E CONSULTAS TECNICA' THEN 'CONSULTA TÉCNICA'
        WHEN PCG.GRE_CALCULADA = 'GABINETE DA PRESIDENCIA' THEN 'PRESIDÊNCIA'
        WHEN PCG.GRE_CALCULADA = 'UNIDADE FINANCEIRA E CONTABIL - UFC' THEN 'UFC'
        WHEN PCG.GRE_CALCULADA = 'UNIDADE DE PARCERIA E RELACOES INSTITUCIONAIS - UR' THEN 'UR'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE SERV ACERVO TECNICO E OPERACIONAL, PROTO' THEN 'ACERVO TÉCNICO'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE SERV ACERVO TÉCNICO E OPERACIONAL, PROTO' THEN 'ACERVO TÉCNICO'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE EXPERIENCIA E JORNADA - GEJ' THEN 'GEJ'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE FISCALIZACAO E PROCEDIMENTOS - GFISC' THEN 'GFISC'
        WHEN PCG.GRE_CALCULADA = 'UGI REGISTRO' THEN 'UGI REGISTRO'
        WHEN PCG.GRE_CALCULADA = 'EISI' THEN 'EISI'
        WHEN PCG.GRE_CALCULADA = 'EQUIPE DE OPERACAO, QUALIDADE E SUCESSO DO CLIENTE' THEN 'EQUIPE OPERAÇÃO'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE ASSUNTOS JURIDICOS' THEN 'JURÍDICO'
        WHEN PCG.GRE_CALCULADA = 'GERENCIA DE DES. E EXECUCAO DE PROJETOS - GDEP' THEN 'GDEP'
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'NÃO ATRIBUÍDO' THEN 'Sem distribuição'
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'NAO ATRIBUIDO' THEN 'Sem distribuição'
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'SEM CLASSIFICAÇÃO' THEN 'SEM CLASSIFICAÇÃO'
        WHEN UPPER(TRIM(PCG.GRE_CALCULADA)) = 'SEM CLASSIFICACAO' THEN 'SEM CLASSIFICAÇÃO'
        WHEN PCG.GRE_CALCULADA IS NULL OR TRIM(PCG.GRE_CALCULADA) = '' THEN 'Sem distribuição'
        ELSE PCG.GRE_CALCULADA
    END <> 'SEM CLASSIFICAÇÃO'
 
-- Comentários originais mantidos:
--    AND (@IN_FINISHED IS NULL OR P.FINISHED = @IN_FINISHED)
--    AND P.CD_TP_PROT = @IN_CD_TP_PROT
--    AND (@IN_CD_SUBTP_PROT IS NULL OR P.CD_SUBTP_PROT = @IN_CD_SUBTP_PROT)
--    AND (@IN_CD_TP_STATUS_PROT IS NULL OR SP.CD_TP_STATUS_PROT = @IN_CD_TP_STATUS_PROT)
 
-- delimitar resultado a
-- FETCH FIRST 1000 ROWS ONLY
;
```