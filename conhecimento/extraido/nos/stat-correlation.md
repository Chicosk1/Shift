# Nó `stat_correlation` — Correlação

| | |
|---|---|
| **Tipo (MCP)** | `stat_correlation` |
| **Rótulo na interface** | Correlação |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `correlacao`, `pearson`, `spearman` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

## O que faz

`[CONFIRMADO-MCP]` "Mede a força e a direção da relação entre duas colunas numéricas (Pearson linear
ou Spearman por postos), com n e estatística t. **NÃO implica causalidade.**"
— `describe_node('stat_correlation')`

O aviso sobre causalidade está no próprio texto do contrato, em maiúsculas. É a única ressalva
metodológica que o nó traz.

Mecanismo declarado:

1. **Filtra linhas com ambos os valores presentes.**
2. Calcula o coeficiente (Pearson, ou Spearman sobre os postos).
3. Deriva a estatística t e um **rótulo de força** (Forte / Moderada / Fraca / Nula).

— `describe_node('stat_correlation')`, seção *Transforms disponíveis*

## Quando usar

`[CONFIRMADO-MCP]`
- Verificar se prazo de pagamento anda junto com volume de compras.
- Checar associação entre desconto concedido e inadimplência.
- Explorar relações **antes** de um teste controlado.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_correlation')`. É o nó mais simples da categoria.

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `column_x` | string | **sim** | — | Primeira coluna numérica |
| `column_y` | string | **sim** | — | Segunda coluna numérica |
| `method` | enum | não | `pearson` | `pearson` / `spearman` |
| `output_field` | string | não | `data` | Nome do campo de saída |

**Não há `group_by`.** O nó mede uma correlação global sobre a tabela toda. Correlação por categoria,
filial ou período exige um fluxo por recorte — ou filtrar antes.

**Não há `min_samples`.** Nenhum piso de amostra declarado, em contraste com `stat_drift`,
`stat_woe`, `stat_discrimination`, `stat_transition` e `stat_anomaly`, que todos têm.

## Entradas esperadas

`[CONFIRMADO-MCP]` Duas colunas numéricas na mesma tabela. Linhas em que qualquer uma das duas
estiver ausente são **descartadas** ("filtra linhas com ambos os valores presentes").

## Saídas produzidas

`[CONFIRMADO-MCP]` O coeficiente, o **n**, a **estatística t** e um rótulo de força
(Forte / Moderada / Fraca / Nula).

`[LACUNA]` O contrato **não declara os cortes** que separam Forte de Moderada de Fraca de Nula. Não
há como saber, sem executar, se 0,45 é "Moderada" ou "Fraca" — e o rótulo é o que a maioria das
pessoas vai ler.

`[LACUNA]` O contrato **não declara p-valor**, apenas a estatística t. Sem graus de liberdade
explícitos e sem p, a significância fica a cargo de quem lê.

`[LACUNA]` O contrato não nomeia as colunas de saída.

## Erros comuns / armadilhas

Este nó **não tem seção "Armadilhas conhecidas"** no contrato — o que é notável, porque correlação é
onde a interpretação errada é mais comum. As armadilhas abaixo são consequência do que está
declarado.

`[CONFIRMADO-MCP]` **"NÃO implica causalidade"** é o aviso do próprio contrato. Vale repetir porque o
nó devolve um rótulo verbal ("Forte") que convida à conclusão causal.

`[CONFIRMADO-MCP]` **Descarte silencioso de linhas incompletas.** O `n` de saída revela quantas
sobraram, mas nada compara com o total recebido. Uma coluna 80% nula produz uma correlação calculada
sobre 20% da base, com rótulo de força igualmente confiante. **Confira o `n`.**

`[CONFIRMADO-MCP]` **Sem `min_samples`, correlação de n=4 sai com rótulo.** Duas variáveis
aleatórias com 4 observações produzem coeficiente alto com frequência. O nó não recusa nem ressalva.

`[CONFIRMADO-MCP]` **Pearson é o padrão, e é o menos robusto dos dois.** Pearson mede relação
**linear** e é sensível a outlier; Spearman trabalha por postos. Numa base de varejo com valores
extremos (venda atacado dentro do varejo, devolução grande), Pearson pode reportar relação que é
efeito de dois pontos. `[LACUNA]` O contrato não sugere quando trocar de método — só lista os dois.

`[CONFIRMADO-MCP]` **Sem `group_by`, o paradoxo de Simpson não tem defesa no nó.** Uma correlação
global pode ter sinal oposto à correlação dentro de cada grupo. O contrato não menciona isso, e o nó
não oferece o recorte.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// "Prazo de pagamento correlaciona com volume de compras?"
{"column_x": "prazo_pagamento_dias", "column_y": "volume_compras_mes", "method": "pearson"}
```

## Observações

1. **Uso legítimo declarado: exploração antes de teste.** O contrato posiciona o nó como
   ferramenta de triagem ("explorar relações antes de um teste controlado"), não de conclusão. Manter
   esse enquadramento na skill.
2. **Para medir se uma variável "explica" um desfecho binário, `stat_woe` é o nó certo, não este.**
   Correlação com uma coluna 0/1 é tecnicamente calculável e analiticamente pobre; `stat_woe`
   devolve WoE/IV com classificação de força e detecção de vazamento.
3. Um dos "use quando" do contrato — "associação entre desconto concedido e inadimplência" — é
   justamente um caso onde `stat_woe` daria resposta melhor. Vale registrar como orientação de
   escolha entre nós.
