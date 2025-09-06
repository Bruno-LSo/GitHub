# DATA_ABERTURA - Data de Abertura do Protocolo

## Descrição Geral

A coluna `DATA_ABERTURA` representa a data e hora em que o protocolo foi oficialmente criado no sistema CREANET. Esta data marca o início da contagem de prazos e SLA para atendimento da solicitação.

## Origem dos Dados

### Tabela Principal
- **Tabela**: `DB2INET.TB_PROTOCOLO_CREANET`
- **Coluna de Origem**: `DT_ABERTURA`
- **Tipo de Dado**: TIMESTAMP - Armazena data e hora completas

## Transformação e Formatação

### Processo de Conversão

A query realiza a conversão do TIMESTAMP para formato brasileiro de data:

```sql
-- Na CTE PROTOCOLO_COMPLETO:
TO_CHAR(P.DT_ABERTURA,'DD/MM/YYYY') AS DATA_ABERTURA_FMT,

-- Também calcula dias para uso em cálculos de SLA:
DAYS(P.DT_ABERTURA) AS DAYS_ABERTURA,

-- No SELECT final:
DATA_ABERTURA_FMT AS DATA_ABERTURA
```

### Exemplos Práticos

Transformação dos dados reais:

| DT_ABERTURA (Original) | DATA_ABERTURA (Formatada) | Uso |
|------------------------|---------------------------|-----|
| 2023-06-26 15:20:18.387800 | 26/06/2023 | Data apresentada ao usuário |
| 2023-06-26 23:11:22.212169 | 26/06/2023 | Mesmo dia, horário diferente |
| 2023-06-27 11:15:45.867384 | 27/06/2023 | Dia seguinte |
| 2025-09-05 21:25:12.483565 | 05/09/2025 | Protocolo recente |

## Regras de Negócio

### 1. Cálculo de Dias para SLA

A query converte a data para número de dias usando a função `DAYS()`:

```sql
-- Usado posteriormente no cálculo do SLA:
CASE
    WHEN FINISHED = 1 THEN DAYS_STATUS - DAYS_ABERTURA
    ELSE DAYS_CURRENT - DAYS_ABERTURA
END
```

### 2. Ponto de Partida para Prazos

A `DATA_ABERTURA` é fundamental para:
- Calcular o tempo decorrido desde a abertura
- Determinar se o protocolo está dentro do prazo
- Estabelecer a data estimada de conclusão

### 3. Validação de Consistência

A data de abertura sempre deve ser:
- Anterior ou igual à data atual
- Anterior ou igual à data de qualquer status do protocolo
- Anterior ou igual à data de pagamento (quando aplicável)

## Relacionamentos com Outras Colunas

A `DATA_ABERTURA` interage diretamente com:

1. **DATA_EST_CONCLUSAO**: Calculada somando dias à data de abertura
2. **SLA**: Base para cálculo do tempo de atendimento
3. **CATEGORIA_STATUS**: Determina se está dentro ou fora do prazo
4. **DT_STATUS**: Comparação para calcular tempo de processamento

## Observações Técnicas

- O formato `DD/MM/YYYY` é padrão brasileiro, facilitando a leitura
- A precisão original em TIMESTAMP é mantida na tabela base
- A conversão para `DAYS()` permite cálculos aritméticos eficientes