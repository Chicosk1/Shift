# Nó `stat_forecast` — Previsão de Demanda

| | |
|---|---|
| **Tipo (MCP)** | `stat_forecast` |
| **Rótulo na interface** | Previsão de Demanda |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `previsao`, `forecast`, `demanda`, `holt-winters`, `croston` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

## O que faz

`[CONFIRMADO-MCP]` "Prevê N períodos à frente por série (opcionalmente por SKU), com Holt-Winters
(tendência+sazonalidade), Croston (giro lento) ou média móvel, incluindo intervalo. Base do
planejamento de compra." — `describe_node('stat_forecast')`

Mecanismo declarado:

1. **Reamostra cada série na frequência escolhida (soma), preenchendo lacunas com 0.**
2. Ajusta o método escolhido e projeta o horizonte com intervalo (**±z·desvio dos resíduos**).
3. Produz uma linha por (série × período previsto).

— `describe_node('stat_forecast')`, seção *Transforms disponíveis*

O passo 1 é importante e é o oposto do `stat_reorder`: aqui o preenchimento com zero é
**automático**, não depende de informar `date_column` (que é obrigatória de todo jeito).

## Quando usar

`[CONFIRMADO-MCP]`
- Prever a demanda de produtos para planejar compra.
- Antecipar sazonalidade e tendência de curto prazo.
- **Alimentar o nó de Ponto de Pedido (`stat_reorder`) com a demanda prevista.**

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_forecast')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `date_column` | string | **sim** | — | Coluna de data da série |
| `value_column` | string | **sim** | — | Coluna de valor a prever (ex.: `quantidade_vendida`) |
| `group_by` | array | não | vazio = série única | Colunas de série (ex.: `[produto_id]`) |
| `frequency` | enum | não | `W` | `D` diária / `W` semanal / `M` mensal |
| `horizon` | number | não | `4` | Nº de períodos a prever |
| `method` | enum | não | `auto` | `auto` / `holt_winters` / `ses` / `croston` / `sma` |
| `seasonal_periods` | number | não | pela frequência: `D=7`, `W=52`, `M=12` | Tamanho do ciclo sazonal para Holt-Winters |
| `max_series` | number | não | `200` | **Teto de séries processadas, as de maior volume** |
| `output_field` | string | não | `data` | Nome do campo de saída |

Note que `ses` aparece no enum de `method` mas **não é descrito** na frase do "o que faz" — que cita
apenas Holt-Winters, Croston e média móvel. `[LACUNA]` O contrato não explica o que `ses` faz nem
quando escolhê-lo (a sigla usual é *simple exponential smoothing*, mas isso é inferência, não
contrato).

## Entradas esperadas

`[CONFIRMADO-MCP]` Histórico de movimento: uma coluna de data, uma coluna de valor e, opcionalmente,
as colunas que definem a série. As lacunas de calendário **não precisam** estar preenchidas — o nó
reamostra e preenche com 0.

## Saídas produzidas

`[CONFIRMADO-MCP]` "Uma linha por (série × período previsto)", com o valor previsto e um intervalo
calculado como **±z·desvio dos resíduos**.

`[LACUNA]` O contrato **não nomeia** as colunas de saída, **nem declara qual `z`** é usado no
intervalo — ou seja, qual nível de confiança o intervalo representa. Não há parâmetro para
configurá-lo. Isso importa: o intervalo é o insumo que dimensiona risco de ruptura, e o nível dele
é desconhecido.

## Erros comuns / armadilhas

Este nó **não tem seção "Armadilhas conhecidas"** no contrato. O que segue vem do que está
declarado, e as consequências silenciosas estão marcadas.

`[CONFIRMADO-MCP]` **`max_series` = 200 corta silenciosamente, e corta pelas menores.** O contrato
diz: "Teto de séries processadas, **as de maior volume**". Num supermercado com 20 mil SKUs, o
padrão devolve previsão para 200 e **omite 19.800** — sem erro. As omitidas são justamente as de
giro lento, que são as que mais precisam de Croston. Quem alimentar o `stat_reorder` com essa saída
recebe ponto de pedido para 200 itens e acha que cobriu o sortimento.

`[CONFIRMADO-MCP]` **`seasonal_periods` padrão para `W` é 52.** Holt-Winters com ciclo de 52
semanas precisa de histórico que cubra o ciclo — tipicamente mais de um ciclo completo. `[LACUNA]`
O contrato **não declara** quanto histórico mínimo o método exige, nem o que o nó faz quando a série
é mais curta que o ciclo (cai para outro método? erra? devolve algo?). Com `method: "auto"` isso é
ainda mais opaco: `[LACUNA]` **o critério de escolha do `auto` não está declarado em nenhum lugar
do contrato.**

`[CONFIRMADO-MCP]` **Preencher lacuna com 0 é decisão semântica.** Para venda, período sem
movimento = demanda zero, e o preenchimento está certo. Para uma série que representa *preço* ou
*saldo*, período sem registro **não** é zero — e o nó vai reamostrar somando e preencher com zero de
todo jeito. Use este nó para fluxo (quantidade vendida), não para estado (preço, saldo).

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// "Como prever a demanda semanal por produto para as próximas 8 semanas?"
{"date_column": "data_venda", "value_column": "quantidade", "group_by": ["produto_id"],
 "frequency": "W", "horizon": 8}
```

## Observações

1. **É a metade de cima do único par de nós encadeado que o contrato declara:**
   `stat_forecast` → `stat_reorder`. Os dois contratos se citam mutuamente ("alimentar o nó de
   Ponto de Pedido", "fechar o laço a jusante do nó de Previsão de Demanda").
2. **Croston existe no enum e é o método certo para giro lento** — o que é a maior parte do
   sortimento de um supermercado por contagem de SKU. Mas o `max_series` padrão elimina exatamente
   essas séries antes de o método ter chance de rodar. Subir `max_series` é pré-requisito para o
   Croston fazer sentido no varejo.
3. `[LACUNA]` Nada no contrato sobre custo/tempo de execução em função de `max_series` ou do número
   de pontos — ao contrário do `stat_anomaly`, que declara o custo da recursão. Subir `max_series`
   para 20 mil é um risco de desempenho não quantificado.
