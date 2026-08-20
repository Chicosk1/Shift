# Nó `if_node` — IF (Row Partition)

| | |
|---|---|
| **Tipo (MCP)** | `if_node` |
| **Rótulo na interface** | IF (Row Partition) |
| **Categoria** | `logic` (grupo *Lógica*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `if` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

> **Este é o nó mais importante do lote de guardrails.** `decision_mode: 'single'` é a única
> primitiva confirmada da plataforma que transforma uma condição em **decisão de lote** — e é
> sobre ela que se apoiam o dry-run e o disjuntor descritos em
> [`../padroes-de-guardrail.md`](../padroes-de-guardrail.md).

> **Fonte única.** Não existe aula nem página de documentação para este nó. Verificado:
> nenhuma ocorrência de `gate mode`, `decision_mode` ou `branch_refs` em
> `conhecimento/bruto/docs/**` nem em `conhecimento/bruto/transcricoes/**`; não há
> `docs/guias-de-uso/nos/if*.md`. Tudo abaixo vem de `describe_node('if_node')`.

## O que faz

`[CONFIRMADO-MCP]` Particiona linhas do dataset em **dois ramos** (`true`/`false`) com base em
condições SQL. Em **gate mode** (sem upstream tabular), avalia as condições contra *metadata* e
**ativa um único ramo**. — `describe_node('if_node')`

São, portanto, **dois nós dentro de um**, e o que decide qual dos dois você tem não é uma opção de
configuração — é **o que está ligado na entrada**:

| Situação do upstream | Comportamento |
|---|---|
| Upstream **tabular** (DuckDB) | Row partition: reparte linhas entre os ramos `true` e `false` |
| Upstream **não tabular** (metadata) | **Gate mode**: avalia contra metadata e ativa **um único ramo** |

`[LACUNA]` O contrato não define o que conta como "upstream tabular" na fronteira: um nó que
devolve `sql_result`/`loop_result` (e não um dataset DuckDB) cai em gate mode ou em row partition?
Falta descobrir a regra exata, porque ela muda o significado do fluxo sem mudar a configuração do
nó.

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('if_node')`
- Dividir dataset em linhas que atendem e não atendem uma condição.
- Executar lógica condicional baseada em **variáveis** (gate mode).
- Redirecionar registros para ramos diferentes do fluxo.

O segundo item é o que interessa ao piloto: *"lógica condicional baseada em variáveis"* é
literalmente o caso de uso do dry-run.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('if_node')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `conditions` | array | **sim** | — | `[{field, operator, value}]` **ou** `[{left: PV, operator, right: PV}]` |
| `logic` | enum | não | `and` | `and` / `or` |
| `decision_mode` | enum | não | `per_row` | `per_row` / `single` |

### `decision_mode` — a peça central

`[CONFIRMADO-MCP]` Citado literalmente, porque é a base de dois dos quatro guardrails:

> `decision_mode` (enum) (opções: per_row, single): 'per_row' (padrão): reparte cada linha entre
> V/F. 'single': avalia SÓ a 1ª linha e envia a tabela inteira por um único ramo (decisão de lote).

Lido com cuidado, isso quer dizer:

| Modo | Quantas linhas são avaliadas | Para onde vão as linhas |
|---|---|---|
| `per_row` (padrão) | todas | cada linha para o ramo dela — **os dois ramos podem receber dados** |
| `single` | **só a 1ª** | **a tabela inteira** por um ramo só — o outro recebe **zero linha** |

Consequências práticas:

1. **`single` é decisão de lote, não de linha.** Serve para "este fluxo pode gravar?" e para
   "quantos itens seriam alterados?". Não serve para teto por item.
2. **`per_row` é o teto por item.** Serve para "esta linha varia demais?" — as linhas que passam
   seguem gravando, as que estouram vão para o outro ramo.
3. **`single` depende da 1ª linha ser representativa.** Quando o campo de decisão vem de uma
   variável do fluxo ou de um `combine` em modo cruzado com uma tabela de 1 linha, o valor é
   uniforme em toda a tabela e a 1ª linha representa o lote. Quando o campo varia por linha,
   `single` decide o lote inteiro **pelo primeiro registro que aparecer** — e a ordem das linhas
   não é garantida sem um `sort` antes. `[INFERIDO]` Nenhum ordenamento é prometido pelo contrato;
   quem usa `single` sobre coluna não uniforme está apostando na ordem física do dataset.

### Formato das condições

`[CONFIRMADO-MCP]` Duas formas aceitas:

- **Forma simples:** `{field, operator, value}` — compara uma coluna com um literal.
- **Forma PV:** `{left: PV, operator, right: PV}` — os dois lados são *ParameterValue*.

`[LACUNA]` O contrato do `if_node` **não descreve o formato do PV**. O único lugar onde `PV` é
detalhado é `describe_node('sql_script')`, cujos exemplos usam
`{"mode": "variable", "variable": "ID"}` e `{"mode": "dynamic", "template": "{{pedido_id}}"}`.
`[INFERIDO]` É plausível que o `if_node` aceite o mesmo `mode: 'variable'` no `left` — o que
permitiria testar uma variável do fluxo **sem passar por coluna**. **Não confirmado, e o
guardrail não deve depender disso**: o caminho confirmado é levar a variável para uma coluna com
`mapper` (ver abaixo) e usar a forma simples.

### Operadores

`[LACUNA]` O contrato do `if_node` **não enumera os operadores**. O que existe de enumeração está
no nó irmão: `describe_node('filter')` cita *"operadores SQL (eq, gt, in, is_null, contains,
etc.)"* e o exemplo usa `is_not_null`. `[INFERIDO]` `lte`/`gte`/`lt`/`neq` provavelmente existem,
mas **nenhum deles está confirmado por escrito**.

> **Recomendação para o piloto:** não aposte em `lte`. Calcule a comparação num nó `math`
> (`variacao_abs <= 15` devolve booleano no DuckDB) e teste no `if_node` com `operator: 'eq'` e
> `value: true`, que é o único operador que aparece nos exemplos do próprio contrato. Custa um nó
> a mais e elimina a dependência de um vocabulário não documentado.

`[DIVERGÊNCIA DE NOMENCLATURA]` `filter` documenta a chave como **`op`**
(`{"field": "status", "op": "eq", "value": "ativo"}`); `if_node` e `assert` documentam
**`operator`**. São nós diferentes com chaves diferentes para a mesma coisa. Não misture.

## Entradas esperadas

Um upstream tabular (row partition) **ou** um upstream de metadata (gate mode). É a entrada, e não
a configuração, que escolhe o regime — ver *O que faz*.

Para decidir sobre uma **variável do fluxo**, o caminho confirmado é trazer a variável para dentro
do dataset antes:

`[VÍDEO]` A aula de ID Sequencial mostra exatamente essa manobra, com outro objetivo. O instrutor
cria uma **variável de workflow booleana** (`reset_idpess`) pelo painel *Variáveis do Workflow*,
com valor padrão, e a mapeia como **coluna** dentro de um nó de Mapeamento
(*"vou adicionar aqui no mapeamento uma coluna de reset... variáveis reset idpes"*). Na execução,
um modal pede o valor (*"deseja reiniciar o idpes? ... vou colocar como sim"*) e a coluna chega
`true` em todas as linhas. A decisão em si ele resolveu com `CASE WHEN` numa expressão de
`mapper`, não com `if_node` — mas a **metade de trazer a variável para uma coluna** é observada e
funciona. — `modulo-3-parte-3H-id-sequencial.txt:77-108`

## Saídas produzidas

`[CONFIRMADO-MCP]` Dois ramos: `true` e `false`.

`[LACUNA]` **A pergunta mais importante que ficou sem resposta.** Em row partition, quando um ramo
recebe **zero linha**, o contrato não diz se os nós daquele ramo:
- (a) **não são executados** (ramo desativado), ou
- (b) **são executados com dataset vazio**.

Isso decide se o dry-run é seguro. Em (a), tudo bem. Em (b), um `bulk_insert` com zero linha
provavelmente é inofensivo, mas um `truncate_table` **não é** — o contrato dele diz que ele
*"limpa uma tabela SQL de destino via TRUNCATE ou DELETE e repassa a referência DuckDB upstream
intacta"*, ou seja, o efeito dele **não depende de haver linha nenhuma**.
Falta descobrir: executar um fluxo de teste com `if_node` + ramo vazio e observar o painel de
execução. **Até então: nunca coloque `truncate_table` num ramo de `if_node`.**

Em gate mode o contrato é explícito (*"ativa um único ramo"*), então lá o problema não existe —
o que reforça gate mode como o regime preferido para o disjuntor, se ele for alcançável.

## Erros comuns

`[INFERIDO]` `conditions` é obrigatório e não tem padrão — nó salvo sem condição deve falhar na
execução. Não confirmado qual é a mensagem.

`[INFERIDO]` Usar `decision_mode: 'single'` com um campo que **varia por linha** não gera erro
nenhum: o nó decide o lote inteiro pela 1ª linha e devolve um resultado plausível e errado. É a
mesma classe de armadilha que o contrato do `stat_transition` documenta explicitamente para outro
nó. Não há aviso do validador.

`[LACUNA]` Não se sabe o que acontece quando o campo de `conditions` **não existe** no upstream —
erro na hora de salvar, erro na execução, ou tudo `false`.

## Exemplos

`[CONFIRMADO-MCP]` O único exemplo do contrato:

```json
// "Como separar clientes ativos dos inativos?"
{"conditions": [{"field": "ativo", "operator": "eq", "value": true}], "logic": "and"}
```

`[INFERIDO]` Decisão de lote para dry-run, na forma que o piloto precisa — combinação de peças
confirmadas, mas a combinação em si não foi executada:

```json
// upstream: mapper que trouxe a variável booleana GRAVAR_DE_VERDADE para a coluna 'gravar'
{"conditions": [{"field": "gravar", "operator": "eq", "value": true}],
 "logic": "and",
 "decision_mode": "single"}
```

`[INFERIDO]` Teto de variação por item — `per_row`, com o booleano pré-calculado num `math`:

```json
// upstream: math com {"target_column": "dentro_do_teto", "expression": "abs(variacao_pct) <= 15"}
{"conditions": [{"field": "dentro_do_teto", "operator": "eq", "value": true}],
 "logic": "and",
 "decision_mode": "per_row"}
```

## Observações para o piloto de margem

1. **`decision_mode: 'single'` é a peça do dry-run e do disjuntor.** É o único mecanismo
   confirmado que faz uma tabela inteira mudar de caminho por causa de uma condição. Sem ele,
   guardrail de lote não existiria na plataforma.
2. **`per_row` é a peça do teto por item.** Os dois guardrails usam o mesmo nó em modos opostos —
   vale nomear os nós no canvas de forma que isso fique óbvio para quem revisar
   (`DISJUNTOR (lote)` vs. `TETO (item)`).
3. **A lacuna do ramo vazio (ver *Saídas*) é o risco aberto do dry-run.** O `if_node` desvia
   linhas; ele não promete *desligar* o ramo. Para uma parada de verdade, o candidato é o nó
   `assert` com `action_on_fail: 'abort'` — ver [`assert`](#) em
   [`../padroes-de-guardrail.md`](../padroes-de-guardrail.md), com a ressalva de que `assert`
   **não aparece em `list_nodes`**.
4. **Não dependa do PV com `mode: 'variable'`** até alguém confirmar. O caminho
   `variável → mapper → coluna → if_node` tem metade confirmada por vídeo e metade por contrato,
   e é o que deve entrar na skill.
5. **Não dependa de `lte`.** Pré-calcule booleanos no `math` e compare com `eq`.
