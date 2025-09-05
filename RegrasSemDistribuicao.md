## **CAMPO GRE - Condições para 'Sem distribuição':**

### **1. Protocolo pago mas sem dados de distribuição**
```sql
WHEN NR_BOLETO IS NOT NULL AND DT_PAGAMENTO IS NOT NULL
    AND COALESCE(DEPARTMENT, '') = ''
    AND COALESCE(GRE_DESC, '') = ''
    AND COALESCE(CIDADE, '') = ''
    AND COALESCE(ATRIBUIDO, '') = ''
THEN 'Sem distribuição'
```
**Explicação:** Quando o protocolo foi pago (tem boleto E pagamento), mas nenhum campo de distribuição foi preenchido.
**Exemplo prático:** Cliente pagou o boleto, mas o sistema não atribuiu a nenhum departamento, cidade ou agente.

### **2. Protocolo não atribuído (que não seja "Sem pagamento")**
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
        ELSE 'Protocolo ainda não atribuído a um agente'  -- <-- ESTE VIRA 'Sem distribuição'
    END
```
**Explicação:** Protocolo sem observações OU sem agente atribuído OU atribuído ao "usuário 0", MAS que não se enquadre como "Sem pagamento".
**Exemplo prático:** Protocolo aberto há 5 dias, sem analista responsável, sem observações, mas dentro do prazo de conclusão.

---

## **CAMPO GRE_Simplificada - Condições ADICIONAIS:**

### **3. Status "Aguardando Pagamento"**
```sql
WHEN DS_STATUS = 'Aguardando Pagamento' 
THEN 'Sem distribuição'
```
**Explicação:** Qualquer protocolo com status "Aguardando Pagamento" automaticamente vira "Sem distribuição".
**Exemplo prático:** Cliente fez a solicitação, boleto foi gerado, mas ainda não foi pago.

### **4. Protocolos não atribuídos (versão simplificada)**
```sql
WHEN COALESCE(DS_OBSERVACOES, '') = '' 
    OR COALESCE(ATRIBUIDO, '') = ''
    OR ATRIBUIDO = '0'
THEN 'Sem distribuição'
```
**Explicação:** Na versão simplificada, TODOS os casos de não atribuição viram "Sem distribuição" (não há a subcondição de "Sem pagamento").
**Exemplo prático:** Qualquer protocolo sem analista ou observações.

### **5. Departamentos explicitamente não atribuídos**
```sql
WHEN UPPER(TRIM(COALESCE(GRE_FROM_MAPPING, DEPARTMENT, ''))) IN ('NÃO ATRIBUÍDO', 'NAO ATRIBUIDO') 
THEN 'Sem distribuição'
```
**Explicação:** Quando o departamento está explicitamente marcado como "NÃO ATRIBUÍDO".
**Exemplo prático:** Sistema marca o departamento como "NÃO ATRIBUÍDO" em vez de deixar vazio.

### **6. Campos de departamento completamente vazios**
```sql
WHEN COALESCE(GRE_FROM_MAPPING, DEPARTMENT, '') = '' 
THEN 'Sem distribuição'
```
**Explicação:** Quando tanto o mapeamento de GRE quanto o departamento estão vazios.
**Exemplo prático:** Protocolo criado mas sem nenhuma informação de unidade organizacional.

---

## **RESUMO TÉCNICO:**

**No campo GRE:** 2 situações principais
**No campo GRE_Simplificada:** 6 situações principais (mais abrangente)

A diferença principal é que **GRE_Simplificada é mais restritiva** - ela transforma mais casos em "Sem distribuição", incluindo todos os protocolos aguardando pagamento e todos os não atribuídos, sem exceções.