# Nó `combine` — Combinar (Union / Join)

| | |
|---|---|
| **Tipo (MCP)** | `combine` — nenhum alias declarado em `describe_node('combine')` |
| **Rótulo na interface** | Combinar `[UI-OBSERVADA]` — `m3-3H:45` |
| **Categoria** | `transform` (grupo *Transformação*) `[CONFIRMADO-MCP]` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | **É o sucessor.** Absorve `union` e `join` — `consolidacao-de-nos-de-transformacao.md` lista `union → combine (config.mode="union")` e `join → combine (config.mode="join")` `[CONFIRMADO-DOC]`. E o próprio contrato de `join` diz *"Versão legada — prefira o nó `combine` com mode='join'"* `[CONFIRMADO-MCP]` |

## O que faz

`[CONFIRMADO-MCP]` Combina múltiplos datasets em um único.

- **modo `union`:** empilha linhas de N entradas.
- **modo `join`:** cruza dois datasets via SQL JOIN — `inner` / `left` / `right` / `full` / **`cross`**.

— `describe_node('combine')`

`[CONFIRMADO-MCP]` O `cross` é o diferencial frente ao nó legado `join`, cujo contrato só oferece
inner/left/right/full.

> `[VÍDEO]` Em `m3-3H` o instrutor **tentou o nó `Junção (Join)` e desistiu**: *"Acho que é o full
> join. Acho que vai dar certo. Ah, condição right, left... não, não é inner, não, não é esse aqui
> que eu quero. Então é o combinar que tem o cross."* — `m3-3H:41-44`
> Em seguida arrastou o nó **Combinar** e o configurou no modo **"Cruzar"**, tipo
> **"Cross join cartesianos"**. — `m3-3H:45,47`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('combine')`
- Unir dados de múltiplas fontes em uma só tabela (union).
- Cruzar dois datasets por colunas em comum (join).
- Adicionar coluna de origem ao dataset unido.

`[CONFIRMADO-DOC]` A tabela "quando usar o consolidado vs. o legado" recomenda:
- **Novo workflow sendo criado** → nós consolidados.
- **Workflow legado em produção** → *não migre*, o legado continua suportado.
- **Empilhamento de N datasets (>2 fontes)** → `combine` modo `union`.
- **Cruzamento de dois datasets com chave** → `combine` modo `join`.
— `consolidacao-de-nos-de-transformacao.md`

`[INFERIDO]` Para **produto cartesiano** (colar uma linha única — um `MAX(id)`, uma cotação, um
fator de correção — em todas as linhas de outro dataset) o `combine` com `join_type: cross` é o
único caminho nativo. É exatamente o uso de `m3-3H`.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` No nível do nó existem só dois campos — `describe_node('combine')`:

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `config` | object | **sim** | Objeto com `mode` (`union` ou `join`) e os campos específicos do modo |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

O `describe_node` **não detalha o interior de `config`** — o detalhamento abaixo vem do documento
de consolidação e dos dois exemplos do próprio contrato.

### `config` no modo `union`

`[CONFIRMADO-DOC]` — `consolidacao-de-nos-de-transformacao.md`

| Campo | Descrição |
|---|---|
| `mode` | Fixo em `"union"` — é o **discriminador de operação** |
| `align` | `by_name` ou `by_position` — o alinhamento de colunas |
| `add_source_col` | Adiciona coluna identificando o handle de origem |
| `source_col_name` | Nome dessa coluna (ex.: `origem`) |
| `dedup_keys` | Colunas-chave da deduplicação pós-união |
| `dedup_priority` | Critério de desempate |

> ⚠️ `[CONFIRMADO-DOC]` **Renomeação que engana.** Citado do documento, porque é a pegadinha nº 1
> da migração:
>
> > "o campo `mode` do nó legado (`"by_name"` / `"by_position"`) vira `align` no novo nó. O campo
> > `mode` do novo nó é sempre `"union"` (discriminador de operação)."
>
> Ou seja: `{"mode": "by_name"}` no `union` legado vira `{"mode": "union", "align": "by_name"}` no
> `combine`. Copiar o JSON velho sem essa troca produz um `mode` inválido.
> — `consolidacao-de-nos-de-transformacao.md`

`[CONFIRMADO-MCP]` O exemplo do próprio contrato confirma o `align`:
`{"config": {"mode": "union", "align": "by_name"}}` — `describe_node('combine')`

**Handles de entrada:** `[CONFIRMADO-DOC]` dinâmicos — `input_1`, `input_2`, …, `input_N`.
— `consolidacao-de-nos-de-transformacao.md`

### `config` no modo `join`

`[CONFIRMADO-DOC]` / `[CONFIRMADO-MCP]`

| Campo | Descrição |
|---|---|
| `mode` | Fixo em `"join"` |
| `join_type` | `inner`, `left`, `right`, `full` **ou `cross`** `[CONFIRMADO-MCP]` |
| `conditions` | Lista `[{left_column, right_column}]` |
| `columns` | Colunas a selecionar; `null` = todas |

**Handles de entrada:** `[CONFIRMADO-DOC]` **fixos `left` e `right`**.
— `consolidacao-de-nos-de-transformacao.md`

> `[UI-OBSERVADA]` **Divergência de nomenclatura.** Na tela do nó legado de junção os handles são
> rotulados **"FROM"** e **"JOIN"** (`m3-3J:45`), e no `combine` o modo aparece como **"Cruzar"**
> com tipo **"Cross join cartesianos"** (`m3-3H:47`). O contrato usa `left`/`right` e
> `mode`/`join_type`. São os mesmos campos com vocabulário diferente.
> `[LACUNA]` Falta confirmar se o `combine` também rotula os handles como FROM/JOIN na tela, ou
> se usa Esquerda/Direita.

`[LACUNA]` Nenhuma fonte mostra o **modo `union` do `combine` na tela**. Todo o material de aula
sobre empilhamento usa o nó legado `União` (`m3-3I`). Falta descobrir se o rótulo do modo é
"Unir"/"União" e se o botão "+" de adicionar entradas se comporta igual.

## Entradas esperadas

- **Modo union:** N datasets upstream (`input_1..input_N`). `[CONFIRMADO-DOC]`
  `[INFERIDO]` O limite mínimo de 2 entradas documentado para o `union` legado provavelmente vale
  aqui — não confirmado.
- **Modo join:** exatamente dois datasets, nos handles `left` e `right`. `[CONFIRMADO-DOC]`

## Saídas produzidas

`[CONFIRMADO-MCP]` O dataset combinado em `output_field` (padrão `data`).

> `[VÍDEO]` **Comportamento a verificar.** Em `m3-3H:59` o instrutor afirma: *"aqui o nó Combinar,
> quando você passa uma tabela que tem mais de um dado ele vai pegar sempre o primeiro"* — falando
> de um `MAX(id)` colado via cross join e consumido pelo nó `ID Sequencial`.
> `[LACUNA]` A frase é ambígua: o "pega sempre o primeiro" pode ser do `combine` **ou** do campo
> "INICIAR EM = Campo" do nó `ID Sequencial` (`m3-3H:52`). Nada no contrato do `combine` menciona
> esse comportamento. Falta descobrir de quem é a regra — importa, porque é o mecanismo de
> sequenciamento de ID em carga incremental.

## Erros comuns

`[CONFIRMADO-DOC]` Reaproveitar o JSON do `union` legado sem trocar `mode` por `align` — ver o
aviso acima.

`[INFERIDO]` Usar `join_type: cross` sem perceber que o resultado é `N × M` linhas. O cross é
seguro quando um dos lados tem **uma linha** (é o caso de `m3-3H`); com dois lados grandes explode.

`[LACUNA]` Não há registro de mensagens de erro reais do `combine` em nenhuma fonte — nem quantos
handles ele exige por modo, nem o que acontece se `conditions` vier vazio em `mode: join`.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato — `describe_node('combine')`:

```json
// Empilhar dois datasets
{"config": {"mode": "union", "align": "by_name"}}

// Cruzar clientes com pedidos pelo id_cliente
{"config": {"mode": "join", "join_type": "inner",
            "conditions": [{"left_column": "id_cliente", "right_column": "id_cliente"}]}}
```

`[CONFIRMADO-DOC]` Migração completa vinda do legado — `consolidacao-de-nos-de-transformacao.md`:

```json
// union legado → combine
{"type": "combine", "output_field": "data",
 "config": {"mode": "union", "align": "by_name", "add_source_col": true,
            "source_col_name": "origem", "dedup_keys": ["id"],
            "dedup_priority": [{"source": "input_1", "priority": 1}]}}

// join legado → combine
{"type": "combine", "output_field": "data",
 "config": {"mode": "join", "join_type": "left",
            "conditions": [{"left_column": "cliente_id", "right_column": "id"}],
            "columns": null}}
```

> ⚠️ **Duas fontes discordam sobre `dedup_priority`.** O documento de consolidação mostra um
> **array de objetos** `[{"source": "input_1", "priority": 1}]` `[CONFIRMADO-DOC]`; o
> `describe_node('union')` declara um **enum** de valores `first` / `last` / `input_first` /
> `input_last` `[CONFIRMADO-MCP]`. Registrado nos dois formatos, sem escolha. Ver `union.md`.

## Observação para o piloto

1. **É aqui que o pedido encontra a margem cadastrada.** `combine` em `mode: join`,
   `join_type: left`, `conditions` casando a chave de produto do pedido com a chave da tabela de
   margem. `left` (e não `inner`) para que um produto **sem** margem cadastrada não desapareça em
   silêncio — ele passa com NULL e pode ser desviado para rejeição.
2. **`cross` para constantes de fluxo.** Fator de reajuste, teto de desconto, data de corte: uma
   linha única colada em todas via `join_type: cross`, no padrão de `m3-3H`.
3. **Migrar ou não.** O documento de consolidação é explícito: fluxo legado em produção **não
   deve** ser migrado só por migrar. Para o piloto, que é novo, começar direto em `combine` evita
   ficar preso ao `join` sem `cross`.
4. **O `output_summary` do `combine` é desconhecido.** O `union` legado documenta `row_count_in`
   por handle e o warning `schema_drift`; nada equivalente foi confirmado para o `combine`. Se a
   automação de preço depender de conferir "entraram X linhas, saíram Y", isso precisa ser testado
   antes. `[LACUNA]`
