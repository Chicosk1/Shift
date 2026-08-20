# Nó `switch_node` — Switch (Partição por Valor)

| | |
|---|---|
| **Tipo (MCP)** | `switch_node` |
| **Rótulo na interface** | Switch (Partição por Valor) |
| **Categoria** | `logic` (grupo *Lógica*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `switch`, `row_partition_switch` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

> **Fonte única.** Não existe aula nem página de documentação para este nó (não há
> `docs/guias-de-uso/nos/switch*.md`; nenhuma menção nas transcrições). Tudo abaixo vem de
> `describe_node('switch_node')`.

## O que faz

`[CONFIRMADO-MCP]` Particiona linhas do dataset entre **N ramos** com base no **valor de um
campo**. Funciona também em **gate mode** quando o upstream é metadata (sem DuckDB): nesse caso
*"ativa exatamente um handle"*. — `describe_node('switch_node')`

É o `if_node` generalizado: mesma dualidade row-partition/gate mode, mesma opção
`decision_mode`, mas em vez de dois ramos fixos (`true`/`false`) você declara os ramos pelos
valores que espera encontrar na coluna.

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('switch_node')`
- Rotear linhas para ramos diferentes dependendo de um campo categórico.
- Separar pedidos por status (pendente, aprovado, cancelado) para processamento distinto.
- Ativar um único caminho do fluxo com base em uma condição.

Regra prática: use `if_node` quando a decisão é binária e expressa uma **condição**; use
`switch_node` quando a decisão é **um valor entre vários** e você não quer uma escada de `if_node`
encadeados.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('switch_node')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `switch_field` | string | **sim** | — | Coluna/chave cujo valor determina o ramo |
| `cases` | array | **sim** | — | `[{label: <handle>, values: [<v1>, ...]}]` |
| `decision_mode` | enum | não | `per_row` | `per_row` / `single` |

### `cases` — um rótulo, vários valores

`[CONFIRMADO-MCP]` Cada caso é um par `label` + lista de `values`. O `label` **é o nome do
handle** de saída no canvas, e vários valores podem cair no mesmo ramo — o exemplo do próprio
contrato usa isso para tratar diferença de caixa (`["aprovado", "APROVADO"]`).

`[CONFIRMADO-MCP]` **Existe um ramo `default`.** O contrato só o menciona ao descrever o
`decision_mode`: *"envia a tabela inteira por um único case (**ou default**)"*.

`[LACUNA]` O `default` não aparece na lista de parâmetros. Falta descobrir: ele é um handle
sempre presente no canvas, ou precisa ser declarado como um `case` de nome `default`? E o que
acontece com uma linha cujo valor não casa com nenhum `case` **se** o `default` não estiver
ligado a nada — a linha é descartada em silêncio ou o nó falha? Isso importa: descarte silencioso
num fluxo de preço é perda de item sem aviso.

`[LACUNA]` O contrato não diz se a comparação de `values` é sensível a caixa e acento. O exemplo
listar `"aprovado"` e `"APROVADO"` separadamente é **indício forte** de que é
sensível a caixa `[INFERIDO]`, mas não é uma afirmação.

### `decision_mode`

`[CONFIRMADO-MCP]` Idêntico ao do `if_node`:

> 'per_row' (padrão): reparte cada linha entre os cases. 'single': avalia SÓ a 1ª linha e envia a
> tabela inteira por um único case (ou default) — decisão de lote.

Ver [`if-node.md`](if-node.md) para a leitura detalhada dos dois modos e para a armadilha de usar
`single` sobre coluna que varia por linha — vale igual aqui.

## Entradas esperadas

Upstream tabular (partição de linhas) ou upstream de metadata (gate mode). Como no `if_node`, é a
entrada que escolhe o regime, não a configuração.

## Saídas produzidas

`[CONFIRMADO-MCP]` Um handle por `case`, mais o `default`.

`[LACUNA]` Vale aqui a mesma pergunta aberta do `if_node`, e ela é pior num nó de N ramos: quando
um handle recebe **zero linha**, os nós daquele ramo são pulados ou executados com dataset vazio?
Sem essa resposta, não se pode afirmar que um ramo não escolhido é inerte. Falta descobrir por
execução observada.

## Erros comuns

`[INFERIDO]` `switch_field` apontando para coluna inexistente — não confirmado se falha ao salvar
ou na execução.

`[INFERIDO]` Valor sem `case` e sem `default` ligado: candidato a perda silenciosa de linhas (ver
lacuna acima). Até confirmar, trate como armadilha real e **sempre ligue o `default`**, mesmo que
seja num `csv_export` de sobra, só para não perder registro.

`[INFERIDO]` `single` com coluna heterogênea: decide o lote pelo primeiro registro, sem erro
nenhum.

## Exemplos

`[CONFIRMADO-MCP]` O único exemplo do contrato:

```json
// "Como separar pedidos por status em ramos distintos?"
{"switch_field": "status",
 "cases": [{"label": "aprovado", "values": ["aprovado", "APROVADO"]},
           {"label": "pendente", "values": ["pendente"]}]}
```

`[INFERIDO]` Aplicado ao piloto — um modo de operação de três estados, em vez de um booleano de
dry-run:

```json
// upstream: mapper que trouxe a variável de texto MODO ('simular'|'aplicar') para a coluna 'modo'
{"switch_field": "modo",
 "cases": [{"label": "simular", "values": ["simular"]},
           {"label": "aplicar", "values": ["aplicar"]}],
 "decision_mode": "single"}
```

## Observações para o piloto de margem

1. **É o `if_node` com mais de dois caminhos.** Para o dry-run, um booleano em `if_node` basta e é
   mais simples de auditar. O `switch_node` só ganha quando o piloto tiver três ou mais modos de
   operação (por exemplo `simular` / `aplicar_parcial` / `aplicar`), ou quando o roteamento for por
   faixa de variação (`ok` / `revisar` / `bloquear`) em vez de por um sim/não.
2. **O `default` é um risco não documentado.** Num fluxo que altera preço, uma linha que cai em
   ramo nenhum é um item que não foi processado **e não foi reportado**. Ligue o `default` sempre.
3. **Mesma lacuna do ramo vazio.** Um `switch_node` não substitui uma parada; ele roteia. A parada
   depende de `assert` com `action_on_fail: 'abort'` — ver
   [`../padroes-de-guardrail.md`](../padroes-de-guardrail.md).
