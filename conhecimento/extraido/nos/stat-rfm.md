# Nó `stat_rfm` — Segmentação RFM

| | |
|---|---|
| **Tipo (MCP)** | `stat_rfm` |
| **Rótulo na interface** | Segmentação RFM |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `rfm`, `segmentacao`, `recencia frequencia valor` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

## O que faz

`[CONFIRMADO-MCP]` "Agrega transações por cliente em Recency, Frequency e Monetary, pontua cada
eixo em quantis e rotula segmentos (Campeões, Fiéis, Em risco, Perdidos...). Base de CRM, campanha
e retenção." — `describe_node('stat_rfm')`

Mecanismo declarado:

1. Agrupa por cliente: `recency` (dias até `reference_date`), `frequency` (contagem), `monetary` (soma).
2. Pontua cada eixo com **NTILE** (recente / frequente / valioso ⇒ nota alta).
3. Rotula o segmento **a partir das notas R e F**.

— `describe_node('stat_rfm')`, seção *Transforms disponíveis*

Observe o passo 3: o rótulo sai de **R e F apenas**. O eixo **M é calculado e pontuado, mas não
entra no rótulo** — está declarado assim no contrato.

## Quando usar

`[CONFIRMADO-MCP]`
- Segmentar clientes por valor e engajamento.
- Identificar quem está em risco de churn ou já perdido.
- Priorizar campanhas por segmento de comportamento.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_rfm')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `customer_column` | string | **sim** | — | Coluna que identifica o cliente (ex.: `cliente_cnpj`) |
| `date_column` | string | **sim** | — | Coluna de data da transação |
| `value_column` | string | **sim** | — | Coluna de valor monetário da transação |
| `reference_date` | string | não | data máxima do dataset | Data de referência para Recency (AAAA-MM-DD) |
| `tiles` | number | não | `5` | Nº de faixas de pontuação por eixo |
| `output_field` | string | não | `data` | Nome do campo de saída |

## Entradas esperadas

`[CONFIRMADO-MCP]` **Transações** — uma linha por transação, com cliente, data e valor. É o nó que
agrega (ao contrário de `stat_abc`): o contrato diz "agrega transações por cliente".

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma linha por cliente, com recency / frequency / monetary, as notas por eixo e
o rótulo de segmento.

`[LACUNA]` O contrato **não nomeia** as colunas de saída nem lista o **conjunto completo de
segmentos possíveis**. Cita quatro entre reticências: "Campeões, Fiéis, Em risco, Perdidos...".
Quais são os outros, e qual combinação de notas R/F produz cada um, **não está no contrato**.

## Erros comuns / armadilhas

Este nó **não tem seção "Armadilhas conhecidas"** no contrato. O que segue vem do que está
declarado.

`[CONFIRMADO-MCP]` **`reference_date` padrão é a data máxima do dataset, não hoje.** Consequência
direta: se a extração parou de rodar por 40 dias, a recência de todo mundo continua parecendo
recente, porque a régua se move com o dado. Para "quem está em risco **hoje**", informe
`reference_date` explicitamente.

`[CONFIRMADO-MCP]` **O rótulo ignora o eixo M.** Um cliente de ticket altíssimo e compra rara é
rotulado pelo comportamento (R e F), não pelo valor. Quem quiser priorizar por dinheiro precisa
usar a coluna `monetary` da saída, não o rótulo.

`[CONFIRMADO-MCP]` **`tiles` sobre base pequena.** A pontuação é por quantil (NTILE): com 5 faixas
e poucos clientes, as faixas ficam com um punhado de clientes cada e a nota vira ruído. O nó
**não tem `min_samples`** — não há piso de amostra declarado, então ele não recusa nem avisa.

`[LACUNA]` O contrato não declara o que acontece com **empates** no NTILE, nem com cliente que tem
valor **negativo** (devolução) ou **nulo**.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Pergunta: "Como segmentar clientes por RFM?"
{"customer_column": "cliente_cnpj", "date_column": "data_venda", "value_column": "valor_total"}
```

## Observações

1. **É o par natural do `stat_cohort`.** RFM diz *quem* está em risco agora; coorte diz *se a base
   retém* quem entra. Nenhum dos dois substitui o outro, e o contrato não os liga explicitamente.
2. **Varejo de supermercado tem um problema de identificação de cliente** que o nó não resolve:
   `customer_column` exige que a venda esteja atrelada a um cliente identificado. Em supermercado,
   boa parte do cupom é anônimo. `[LACUNA]` Se a base do Viasuper identifica o cliente no cupom
   (cartão fidelidade, CPF na nota) é pergunta de domínio, não respondível pelo MCP.
3. `stat_transition` pode consumir a saída deste nó como snapshot mensal para medir migração entre
   segmentos — o contrato do `stat_transition` cita "faixa de compra" nesse papel.
