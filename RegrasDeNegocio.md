#  DOCUMENTO COMPLETO DE EXIGÊNCIAS - VERSÃO ATUALIZADA

##  **EXIGÊNCIAS JÁ IMPLEMENTADAS NA QUERY ATUAL**

### 1. **EXIGÊNCIA ATENDIDA = ANÁLISE**
**Fonte no documento:**
```
Linha 895: "exigência atendida = Análise"
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
Linhas 510-512: "Status SLA - Aberto e Fechado"
Linhas 574-577: "'cancelado', 'deferido' e 'indeferido' serão agora classificados como 'FECHADO'..."
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
Linhas 507-509: "SLA OFICIAL - Tempo oficial... SLA PROTOCOLO - Tempo que o protocolo levou..."
Linhas 520-554: [Tabela com 34 subtipos e prazos]
Linha 558-559: "Por exemplo alteração de registro de empresa maior que 20 dias de SLA, 
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
Linha 504: "para os casos de Acervo, se o funcionário tiver algum protocolo de Acervo, 
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
Linhas 565-568: "se a solicitação esta cancelada, com status fechada e na observação 
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
Linha 513: "o Login esta como 0, então considerar como não atribuido"
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
Linha 911: "sem pagto muda para: cancelado sem pagto"
Linha 913: "aguardando pagamento vai aparecer como: sem distribuição"
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
Linha 909: "presidência vai sair - serão removidas as linhas"
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
Linhas 901-906: "Os resultados a seguir serão considerados TESTES"
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