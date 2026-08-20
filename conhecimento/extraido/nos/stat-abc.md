# Nó `stat_abc` — Curva ABC (Pareto)

| | |
|---|---|
| **Tipo (MCP)** | `stat_abc` |
| **Rótulo na interface** | Curva ABC (Pareto) |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `pareto`, `curva abc`, `abc` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

> **Este é o único nó de todo o material — MCP incluído — que menciona a palavra "margem".**
> Ver *Observações* no fim, porque a menção é mais estreita do que parece.

## O que faz

`[CONFIRMADO-MCP]` "Classifica cada linha em classes A/B/C pela concentração acumulada de uma
coluna de valor (faturamento, margem, quantidade). Base para foco de estoque, atendimento e
sortimento." — `describe_node('stat_abc')`

O mecanismo declarado é aritmética de Pareto, sem modelo estatístico:

1. Ordena as linhas por `value_column` decrescente.
2. Calcula o percentual acumulado do valor total.
3. Atribui `A` até `class_a_threshold`, `B` até `class_b_threshold`, `C` no restante.

— `describe_node('stat_abc')`, seção *Transforms disponíveis*

## Quando usar

`[CONFIRMADO-MCP]`
- Descobrir os 20% de produtos que geram 80% da receita.
- Priorizar clientes A para atendimento diferenciado.
- Classificar SKUs por relevância antes de decidir reposição.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_abc')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `value_column` | string | **sim** | — | Coluna numérica que mede importância (ex.: `faturamento_total`, `margem`) |
| `id_column` | string | não | — | Coluna identificadora (ex.: `produto_id`). **Usada só como critério de desempate estável** |
| `class_a_threshold` | number | não | `0.8` | Corte acumulado da classe A (80%) |
| `class_b_threshold` | number | não | `0.95` | Corte acumulado da classe B (95%). Acima disso é classe C |
| `output_field` | string | não | `data` | Nome do campo de saída |

Atenção ao papel de `id_column`: o contrato diz explicitamente que ela é **critério de desempate**,
não chave de agregação. O nó **não agrupa** — ele classifica **cada linha** recebida.

## Entradas esperadas

`[CONFIRMADO-MCP]` Um dataset tabular com uma coluna numérica de valor. O contrato descreve a
classificação como aplicada a "cada linha".

`[LACUNA]` O contrato **não diz** se o nó agrega por `id_column` quando há várias linhas do mesmo
produto. Como os transforms declarados são apenas ordenar / acumular / rotular, e `id_column` é
descrita só como desempate, a leitura literal é que **não há agregação** — logo o dataset já deve
chegar somado (uma linha por entidade). Isso **não está afirmado** no contrato e precisa de teste
antes de virar recomendação.

`[LACUNA]` O contrato não declara comportamento para valores **nulos** nem **negativos** em
`value_column`. Valor negativo (devolução, margem negativa) muda o significado do percentual
acumulado, e nada no contrato diz o que acontece.

## Saídas produzidas

`[CONFIRMADO-MCP]` A classe A/B/C por linha e o percentual acumulado do valor total.

`[LACUNA]` O contrato **não nomeia** as colunas de saída (ao contrário de `stat_anomaly`, que
declara `anomalia_score` e `is_anomalia`, ou de `stat_drift`, que declara `psi_total` e
`classificacao`). Os nomes exatos precisam ser observados numa execução.

## Erros comuns / armadilhas

Este nó **não tem seção "Armadilhas conhecidas"** no contrato — diferente de `stat_anomaly`,
`stat_transition`, `stat_drift`, `stat_woe` e `stat_discrimination`. O que segue são consequências
diretas do que o contrato declara, marcadas conforme a confiança.

`[CONFIRMADO-MCP]` **`id_column` não agrupa.** Quem espera que informar `produto_id` faça o nó
somar as vendas por produto está lendo um parâmetro de desempate como parâmetro de agregação. Some
antes, com um nó de agregação.

`[LACUNA]` Sem declaração sobre nulos, negativos, ou volume mínimo de linhas. Note que os outros
nós de estatística declaram `min_samples` / `min_events` e recusam amostra pequena; `stat_abc`
**não tem nenhum piso de amostra** — ele classifica 3 linhas com a mesma naturalidade com que
classifica 300 mil, e o resultado com 3 linhas é aritmeticamente correto e analiticamente vazio.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Pergunta: "Como classificar produtos em curva ABC pelo faturamento?"
{"value_column": "faturamento_total", "id_column": "produto_id"}
```

## Observações

1. **A menção a "margem" é vocabular, não funcional.** O contrato lista `margem` como um dos
   exemplos aceitáveis de `value_column` — junto com `faturamento` e `quantidade`. Ou seja: o nó
   aceita **qualquer** coluna numérica de valor, e "margem" é apenas um dos rótulos citados. O
   contrato **não** diz onde a margem está, como se calcula, nem qual tabela do ERP a contém.
   Isso continua sendo a lacuna **L4**, intocada.
2. **Combinação declarada com `stat_transition`.** O contrato de `stat_transition` cita
   explicitamente "ver a chance de um cliente mudar de faixa (curva ABC, faixa de compra) no
   próximo período" — ou seja, a saída de `stat_abc` por período é entrada natural (como snapshot)
   do `stat_transition`. É a única ligação entre nós de estatística declarada fora do par
   `stat_forecast` → `stat_reorder`.
3. Os dois cortes são configuráveis. O par `0.8` / `0.95` é padrão, não regra fixa — varejo com
   cauda longa de SKU costuma querer cortes diferentes, e o contrato não opina sobre quais.
