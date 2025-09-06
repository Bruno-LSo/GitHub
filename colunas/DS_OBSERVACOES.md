# DS_OBSERVACOES - Observações do Status

## Descrição Geral

A coluna `DS_OBSERVACOES` contém as observações textuais registradas no último status do protocolo, incluindo informações detalhadas sobre o processamento, exigências ou motivos de cancelamento.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_STATUS_PROTOCOLO_CREANET`
- **Coluna de Origem**: `DS_OBSERVACOES`
- **Tipo de Dado**: VARCHAR/TEXT - Texto livre com observações

### Obtenção através da CTE

```sql
-- Na CTE ULTIMO_STATUS:
SELECT 
    SP.CD_PROT,
    SP.DS_OBSERVACOES,
    -- outras colunas...
    ROW_NUMBER() OVER (PARTITION BY SP.CD_PROT ORDER BY SP.CD_STAT_PROT DESC) AS RN
FROM DB2INET.TB_STATUS_PROTOCOLO_CREANET SP

-- Depois utilizada no JOIN:
INNER JOIN ULTIMO_STATUS US ON US.CD_PROT = P.CD_PROT AND US.RN = 1
```

## Tipos de Conteúdo

### 1. Mensagens de Deferimento
Exemplo real do protocolo PR0309572023:
```
"Caso venha a realizar o exercício profissional durante a interrupção do registro, 
estará sujeito à autuação por exercício ilegal da profissão com as penalidades 
previstas na Lei nº 5.194/66 e demais cominações legais..."
```

### 2. Mensagens de Cancelamento por Boleto Vencido
Exemplo real dos protocolos PR0311262023 e PR0311962023:
```
"Boleto vencido: 28027180231412766 - Vencto.: 04/07/2023 00:00:00"
```

### 3. Instruções de Procedimento
Exemplo real do protocolo PR0310732023:
```
"Efetivado o seu registro de nº 5071006015 e solicitada a sua carteira de 
identidade profissional, a qual será enviada pelos Correios..."
```

## Utilização nas Regras de Negócio

### 1. Identificação de Cancelamento por Não Pagamento

A query verifica o conteúdo das observações para categorização:

```sql
-- Em CATEGORIA_STATUS:
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 'Cancelado por não pagamento'

-- Em GRE:
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 'Cancelado por não pagamento'

-- Em GRE_Simplificada:
WHEN DS_STATUS = 'Solicitação Cancelada' 
     AND FINISHED = 1 
     AND DS_OBSERVACOES LIKE '%Boleto Vencido%'
THEN 'Cancelado por não pagamento'
```

### 2. Validação de Atribuição

A query verifica se há observações para determinar distribuição:

```sql
-- Na coluna GRE:
WHEN COALESCE(DS_OBSERVACOES, '') = '' 
    OR COALESCE(ATRIBUIDO, '') = ''
    OR ATRIBUIDO = '0'
THEN 'Protocolo ainda não atribuído a um agente'

-- Na coluna GRE_Simplificada:
WHEN COALESCE(DS_OBSERVACOES, '') = '' 
    OR COALESCE(ATRIBUIDO, '') = ''
    OR ATRIBUIDO = '0'
THEN 'Sem distribuição'
```

## Exemplos Práticos no Resultado

Dados reais mostrando diferentes tipos de observações:

| NR_PROT | DS_STATUS | DS_OBSERVACOES (truncado) | CATEGORIA_STATUS |
|---------|-----------|---------------------------|------------------|
| PR0309572023 | Solicitação Deferida | Caso venha a realizar... | Fora do Prazo |
| PR0310732023 | Solicitação Deferida | Efetivado o seu registro... | Fora do Prazo |
| PR0311262023 | Solicitação Cancelada | Boleto vencido: 280271... | Cancelado por não pagamento |
| PR0311962023 | Solicitação Cancelada | Boleto vencido: 280271... | Cancelado por não pagamento |

## Padrões Identificados

### Boletos Vencidos
Formato padrão: `"Boleto vencido: [NÚMERO_BOLETO] - Vencto.: [DATA] [HORA]"`

### Atribuições
Conforme dados da tabela TB_STATUS_PROTOCOLO_CREANET:
```
"Protocolo atribuído a: [usuario1] por [usuario2]"
```

### Protocolos CREANET
Mensagem padrão para protocolos criados via web:
```
"Protocolo gerado através do CREANET. Verifique a situação de seu protocolo 
pelo menu Solicitações / Acompanhar serviços solicitados"
```

## Observações Técnicas

- Campo de texto livre sem estrutura fixa
- Pode conter caracteres especiais e quebras de linha
- Utilizado para auditoria e rastreabilidade
- Fundamental para identificar motivos de cancelamento
- O operador `LIKE '%Boleto Vencido%'` é case-sensitive no DB2
- COALESCE é usado para tratar valores NULL nas validações

---