# Nó `stat_reorder` — Ponto de Pedido / Estoque de Segurança

| | |
|---|---|
| **Tipo (MCP)** | `stat_reorder` |
| **Rótulo na interface** | Ponto de Pedido / Estoque de Segurança |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `ponto de pedido`, `estoque de seguranca`, `reorder point`, `safety stock` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

> **Único nó da categoria que declara as fórmulas.** Vale citá-las literalmente, porque é a exceção.

## O que faz

`[CONFIRMADO-MCP]` "Transforma a demanda (realizada ou prevista) em decisão de compra: estoque de
segurança e ponto de pedido por item, a partir do nível de serviço e do lead time. **É a
variabilidade, não só a média, que dimensiona o estoque.**" — `describe_node('stat_reorder')`

E o aviso que o contrato coloca já na descrição, não na lista de armadilhas:

> Informe 'date_column' para que os períodos SEM movimento contem como demanda zero — sem isso a
> média sai inflada nos itens de giro lento.

### Fórmulas declaradas

`[CONFIRMADO-MCP]` — seção *Transforms disponíveis*:

1. Soma a demanda por período e **preenche com ZERO os períodos sem movimento** (com `date_column`).
2. Calcula demanda **média** e **desvio-padrão** por item.
3. `estoque de segurança = z * desvio * sqrt(lead_time)`
4. `ponto de pedido = média * lead_time + estoque de segurança`

`[LACUNA]` O `z` vem do `service_level`, mas o contrato **não declara a tabela de conversão** nem se
usa aproximação normal. Para `service_level = 0.95` a leitura normal-padrão daria z ≈ 1,645, mas
isso é inferência estatística comum, **não** afirmação do contrato.

## Quando usar

`[CONFIRMADO-MCP]`
- Definir quando e quanto repor de cada SKU.
- Dimensionar estoque de segurança por nível de serviço.
- **Fechar o laço a jusante do nó de Previsão de Demanda (`stat_forecast`).**

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_reorder')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `demand_column` | string | **sim** | — | Coluna com a demanda por período (ex.: `quantidade_vendida` por dia/semana) |
| `date_column` | string | não (mas ver armadilha) | — | Coluna de data do movimento. Com ela o nó monta o calendário completo e conta período sem venda como demanda **ZERO**. Sem ela, calcula sobre as linhas recebidas **e avisa** |
| `frequency` | enum | não | `day` | `day` / `week` / `month` — período do calendário quando há `date_column` |
| `group_by` | array | não | vazio = um único resultado global | Colunas de item (ex.: `[produto_id]`) |
| `lead_time` | number | não | `7` | Lead time de reposição, **na mesma unidade de período da demanda** |
| `service_level` | number | não | `0.95` | Nível de serviço desejado, 0–1 |
| `output_field` | string | não | `data` | Nome do campo de saída |

## Entradas esperadas

`[CONFIRMADO-MCP]` Demanda por período — "realizada ou prevista". Ou seja, aceita tanto o histórico
de vendas quanto a saída do `stat_forecast`.

`[CONFIRMADO-MCP]` **Sem `date_column`, o nó calcula sobre as linhas que recebeu, como se cada linha
fosse um período.** Isso funciona apenas se o upstream já entregou o calendário completo, com os
zeros.

## Saídas produzidas

`[CONFIRMADO-MCP]` Por item (`group_by`): demanda média, desvio-padrão, estoque de segurança e ponto
de pedido.

`[LACUNA]` O contrato **não nomeia** as colunas de saída.

## Erros comuns / armadilhas

Este nó **não tem seção "Armadilhas conhecidas"** formal, mas embute o aviso principal na própria
descrição — sinal de que é a armadilha que mais aparece.

`[CONFIRMADO-MCP]` **A armadilha central, literal:**

> Informe 'date_column' para que os períodos SEM movimento contem como demanda zero — sem isso a
> média sai inflada nos itens de giro lento.

Mecanismo: um item que vendeu 1 unidade em 3 dias do mês tem média 1,0/dia se o nó só vê as 3
linhas, e média 0,1/dia se vê o calendário de 30 dias. O ponto de pedido resultante fica **10 vezes
maior** do que deveria. É o erro que compra estoque parado.

`[CONFIRMADO-MCP]` **Sem `date_column` o nó "avisa"** — ou seja, esta armadilha, ao contrário da
maioria da categoria, **tem sinalização**. O contrato não diz em que forma (warning na execução?).
`[LACUNA]` Formato e visibilidade do aviso não declarados.

`[CONFIRMADO-MCP]` **`lead_time` está "na mesma unidade de período da demanda", e isso não é
validado.** Se a demanda está reamostrada por semana (`frequency: "week"`) e alguém informa
`lead_time: 7` pensando em dias, o nó calcula com 7 **semanas** de lead time. O resultado é um
número plausível e muito errado, sem erro nenhum — o `sqrt(lead_time)` e o `média * lead_time`
simplesmente escalam. Esta é a armadilha silenciosa mais perigosa do nó, e **não está na lista de
armadilhas do contrato** — decorre da fórmula declarada.

`[CONFIRMADO-MCP]` **`group_by` vazio produz um único resultado global**, não um por item. Ponto de
pedido agregado de todo o sortimento não tem uso prático; é quase sempre erro de configuração.

`[LACUNA]` O nó **não tem `min_samples`**. Não há piso de amostra declarado: um item com 2 períodos
de histórico produz desvio-padrão e ponto de pedido sem qualquer ressalva.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato — note que o segundo exemplo é a resposta a um sintoma, o que
confirma que a armadilha da `date_column` é comum na prática:

```json
// "Como calcular ponto de pedido por produto com 95% de nível de serviço?"
{"demand_column": "quantidade_vendida", "date_column": "data_venda", "frequency": "day",
 "group_by": ["produto_id"], "lead_time": 7, "service_level": 0.95}

// "Minha base tem só os dias em que houve venda. O ponto de pedido está alto demais."
{"demand_column": "quantidade_vendida", "date_column": "data_venda", "frequency": "day",
 "group_by": ["produto_id"]}
```

## Observações

1. **É o nó mais diretamente aplicável a supermercado de toda a categoria** — responde "quanto
   comprar e quando" com fórmula explícita, e o problema de giro lento que ele avisa é exatamente o
   perfil de sortimento de mercado.
2. **Encadeamento declarado:** `stat_forecast` → `stat_reorder`. Cuidado ao usar a saída do forecast
   como entrada: o forecast tem `max_series` padrão **200**, então o reorder herda a amputação do
   sortimento sem saber.
3. `[LACUNA]` O contrato **não trata lead time variável** — `lead_time` é um número único, aplicado a
   todos os itens do `group_by`. Fornecedor com prazo diferente por item exige um fluxo por faixa de
   lead time, ou um nó por fornecedor. Nada no contrato sobre isso.
4. `[LACUNA]` Nada sobre estoque atual, lote mínimo de compra, múltiplo de embalagem ou validade —
   o nó devolve o **ponto de pedido**, não a **quantidade a comprar**. A decisão final de compra
   precisa de dados que o nó não consome.
