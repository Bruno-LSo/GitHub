# NR_PROT - Número do Protocolo

## Descrição Geral

A coluna `NR_PROT` representa o número único de identificação do protocolo no sistema CREANET. Este identificador é fundamental para rastreabilidade e acompanhamento de todas as solicitações realizadas no CREA-SP.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_PROTOCOLO_CREANET`
- **Coluna de Origem**: `NR_PROT`
- **Tipo de Dado**: VARCHAR - Armazena o identificador alfanumérico do protocolo

## Estrutura e Formato

O número do protocolo segue o padrão: `PRXXXXXXXXAAAA`

Onde:
- `PR` - Prefixo fixo indicando "Protocolo"
- `XXXXXXXX` - Sequência numérica do protocolo
- `AAAA` - Ano de abertura

### Exemplo da Query

```sql
-- Na query principal, o NR_PROT é extraído diretamente:
SELECT
    P.NR_PROT,
    -- outras colunas...
FROM DB2INET.TB_PROTOCOLO_CREANET P
```

### Exemplos Práticos

Com base nos dados reais da tabela `TB_PROTOCOLO_CREANET`:

| NR_PROT | Descrição |
|---------|-----------|
| PR0467692023 | Protocolo número 046769 aberto em 2023 |
| PR0309572023 | Protocolo número 030957 aberto em 2023 |
| PR0310732023 | Protocolo número 031073 aberto em 2023 |
| PR0311262023 | Protocolo número 031126 aberto em 2023 |

## Regras de Negócio

1. **Unicidade**: Cada protocolo possui um número único no sistema
2. **Imutabilidade**: Uma vez gerado, o número do protocolo não pode ser alterado
3. **Sequencialidade**: Os números seguem uma sequência controlada pelo sistema
4. **Filtro de Rascunhos**: A query principal filtra apenas protocolos válidos através da condição `WHERE P.DRAFT = 0`, excluindo protocolos em rascunho

## Relacionamentos

O `NR_PROT` é utilizado como chave de relacionamento com:
- `TB_BOLETO` através da coluna `CD_PROC` para vincular informações de pagamento
- Identificação única em relatórios e consultas do BI

## Observações Técnicas

- A coluna é indexada para otimizar consultas (índice `IDX_PROT_DRAFT_FINISHED`)
- Não permite valores nulos
- Serve como identificador principal para o usuário final localizar suas solicitações

---
