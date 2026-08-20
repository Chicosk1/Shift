# Nó `sql_script` — Script SQL

| | |
|---|---|
| **Tipo (MCP)** | `sql_script` |
| **Rótulo no contrato do MCP** | Script SQL |
| **Rótulo na interface (aula)** | **Script** / **SQL Script** — `m3p2b:97,109-110` `[UI-OBSERVADA]` |
| **Categoria** | `database` (grupo *Banco de Dados*) `[CONFIRMADO-MCP]` |
| **Risco** | **`unknown_write`** `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

> **Risco `unknown_write` — categoria própria.** A plataforma **não sabe estaticamente** se o
> script escreve: o mesmo nó aceita `SELECT`, `UPDATE`, `DELETE` e DDL. Nenhum outro nó do catálogo
> tem esse rótulo. Todo o *Guardrails* de `plano-skills-shift.md` §5.4 se aplica com força dobrada
> aqui, porque **o próprio motor não consegue avisar**. Ler `checklist-pre-producao` antes.

## O que faz

`[CONFIRMADO-MCP]` Executa **SQL arbitrário** (`SELECT`/`INSERT`/`UPDATE`/`DELETE`/DDL) com
**parâmetros nomeados** contra um banco de dados externo. Três modos: `query` (retorna dados),
`execute` (afeta linhas) e `execute_many` (itera sobre upstream).
— `describe_node('sql_script')`

`[VÍDEO]` Em aula, a definição em uma frase: *"o nó Script ele foi feito para você **executar
coisas**"*. — `m3p2b:111`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('sql_script')`
- Executar um `SELECT` **parametrizado** e usar o resultado downstream.
- Executar `INSERT`/`UPDATE`/`DELETE` em banco externo com parâmetros do workflow.
- Chamar **stored procedure** ou **DDL** em banco externo.
- Processar **cada linha upstream** contra o banco (`execute_many`).

`[VÍDEO]` A regra de escolha contra o `sql_database`, narrada em aula:

> *"O nó SELECT ele só faz SELECT pronto, ele não tem a passagem de parâmetros, então vale
> ressaltar isso. Então eu não poderia colocar aqui um dois pontos, WHERE dois pontos estab, né,
> pra pra buscar um um estab específico."* — `m3p2b:106`

> ⚠️ **DIVERGÊNCIA — registro das duas fontes, sem escolher lado.**
> `[CONFIRMADO-MCP]` `describe_node('sql_database')` **contradiz isso**: o nó `sql_database` tem
> parâmetro `parameters` ("Mapa de parâmetros `{nome: ParameterValue}` para os placeholders `:nome`
> da query — valor fixo, variável do fluxo ou campo do passo anterior") e um exemplo oficial
> `SELECT * FROM pedidos WHERE cnpj = :cnpj`.
> **Fonte A:** `m3p2b:106` (aula) — *"o SELECT não tem a passagem de parâmetros"*.
> **Fonte B:** `describe_node('sql_database')` (MCP) — tem, com bind real.
> `[INFERIDO]` Explicação plausível: a aula é anterior ao recurso, ou o campo existe no contrato e
> não na UI da versão gravada. **Não confirmado.** Ver `divergencias.md`.
> Consequência prática: **a regra de escolha narrada em aula pode estar obsoleta** — se
> `sql_database` aceita bind, ele passa a ser preferível também para `SELECT` parametrizado, porque
> é `read_only` em vez de `unknown_write` e tem streaming/particionamento.

`[VÍDEO]` A parte da regra que **não** está em disputa: para *executar* algo — `UPDATE`, procedure,
bloco `BEGIN/END` — o `sql_script` é o único caminho. `m3p2b:123-133`

## ⚠️ Armadilhas declaradas no contrato

`[CONFIRMADO-MCP]` Citadas literalmente — `describe_node('sql_script')`:

> - **Placeholder dentro de COMENTÁRIO conta como parâmetro obrigatório:** o SQLAlchemy varre o
>   script inteiro e um `:cnpj` escrito num `-- filtra por :cnpj` vira bind exigido. Ou remova o
>   `:` do comentário, ou forneça o parâmetro — senão a execução falha pedindo um valor que
>   ninguém sabe de onde saiu.
> - **Parâmetro que resolve para `None` é RECUSADO na execução** (o nó levanta antes de rodar o
>   SQL). Todo `:nome` do script precisa de um valor de verdade em `parameters`.
> - **Nunca use `''` para dizer 'sem valor': no Oracle string vazia JÁ É NULL** (é o que o
>   `load_service` documenta), então o filtro muda de sentido em silêncio conforme o banco. Se o
>   valor é opcional, **tire o trecho do SQL** em vez de mandar vazio.

> ⚠️ **A terceira é Oracle-específica e é a mais perigosa do lote.** `WHERE campo = ''` no Oracle
> equivale a `WHERE campo = NULL`, que **nunca é verdadeiro** — o `UPDATE` de preço simplesmente
> não atualiza nada e o nó reporta sucesso com 0 linhas afetadas. Como o escopo do projeto é
> **Oracle**, isto deixa de ser curiosidade e passa a ser regra imperativa da skill.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('sql_script')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `script` | string | **sim** | — | Script SQL com placeholders `:nome` para bindings |
| `mode` | enum | **sim** | — | `query` / `execute` / `execute_many` |
| `connection_id` | string | **sim** | — | UUID do conector SQL de destino |
| `parameters` | object | não | — | Mapa `{nome: ParameterValue}` |
| `output_field` | string | não | `sql_result` | Nome do campo de saída |
| `timeout_seconds` | number | não | **`60`** | Timeout de execução em segundos |
| `output_schema` | array | não | — | Schema declarado de saída `[{name, type}]` para validação |
| `read_delivery_mode` | enum | não | `auto` | `auto` / `tunnel` / `edge` — **só no modo `query`**, para conexões via relay |

> ⚠️ `[CONFIRMADO-MCP]` **`timeout_seconds` padrão é 60 segundos.** É o valor mais apertado entre
> os nós de banco. Um `UPDATE` em volume ou uma procedure de fechamento estoura isso com
> facilidade, e o efeito no banco de um script cortado no meio é `[LACUNA]` — não se sabe se há
> rollback. Ver L37.

## Os três modos

`[UI-OBSERVADA]` Em aula os três aparecem como abas dentro do painel do nó, rotuladas **Query**,
**Execute** e **Exec Many**: *"Aqui dentro a gente tem o Query, o Execute e o Exec Many"*.
— `m3p2b:120-121`

| Modo (MCP) | Rótulo na UI | O que faz `[CONFIRMADO-MCP]` |
|---|---|---|
| `query` | Query | Retorna dados — uma execução |
| `execute` | Execute | Afeta linhas — uma execução |
| `execute_many` | Exec Many | **Itera sobre o upstream** — uma execução por linha |

### A regra que o instrutor errou em aula, ao vivo

`[VÍDEO]` O erro e a correção estão registrados na transcrição e valem mais que a regra abstrata.
O instrutor montou `SELECT * FROM filial WHERE estab = :estab` no modo **Query**, ligando o
parâmetro a um campo do nó anterior, e recebeu **"Estáb não encontrado"** (`m3p2b:117`). Chegou a
inserir um nó de Mapeamento no meio suspeitando do gatilho manual (`m3p2b:118-119`). A causa:

> *"quando eu executo uma query aqui é para quando eu tenho **parâmetros fixos**. (…) Quando eu
> tenho um **parâmetro variável** eu tenho que usar o **Exec Many**. Então isso foi um erro de, de
> esquecimento meu."* — `m3p2b:120-122`

`[INFERIDO]` **Reconciliação precisa, cruzando com os exemplos do MCP** (a formulação da aula é
imprecisa: "variável" ali não quer dizer variável de fluxo):

| Origem do valor do parâmetro | Modo que funciona |
|---|---|
| Valor **fixo** | `query` / `execute` |
| **Variável do fluxo** (`{"mode": "variable", "variable": "ID"}`) | `query` / `execute` — é o exemplo oficial do contrato |
| **Campo do passo anterior** (`{"mode": "dynamic", "template": "{{pedido_id}}"}`) | **`execute_many`** — é o exemplo oficial do contrato |

Ou seja: o que exige `execute_many` é o parâmetro vir de **linha do upstream**, não de "ser
variável". Ambos os exemplos citados são `[CONFIRMADO-MCP]`; o mapeamento entre eles e a fala da
aula é `[INFERIDO]`.

### `execute` aceita bloco `DECLARE / BEGIN / END`

`[VÍDEO]` Demonstrado em aula, contra o ConstruShow:

> *"Esse execute aqui ele também permite você fazer execuções dentro de BEGIN END, então se você
> tem alguma execução mais, eh, digamos, complexa, você também pode fazer aqui pelo execute (…)
> chamar uma função."* — `m3p2b:123`
> *"o nó execute aqui permite que você encadeie essas execuções aqui, né, de declaração, BEGIN END,
> dentro também aqui do Shift."* — `m3p2b:130`

`[VÍDEO]` O objeto usado no exemplo foi `SEQ_PRIMARY_KEY`, do schema do ConstruShow — o próprio
instrutor não tinha certeza se era função ou procedure (`m3p2b:125-130`). Casos de uso narrados:
*"desabilitar alguma trigger, ou fazer uma cadeia de verificação entre tabelas"* (`m3p2b:133`).

`[VÍDEO]` E o comportamento de retorno: *"quando é uma função eu não tenho nenhum return aqui, né,
então por isso que ele não vai retornar nada mesmo. É, porque é para executar alguma coisa."*
— `m3p2b:133`

## Entradas esperadas

`[VÍDEO]` *"Sempre ele vai, aqui no nó anterior ele recebeu uma tabela. (…) Então assim, ele
sempre passa a tabela para o próximo nó."* — `m3p2b:120`

`[CONFIRMADO-MCP]` No modo `execute_many`, o dataset upstream é **obrigatório na prática** — é
sobre ele que o nó itera.

## Saídas produzidas

`[CONFIRMADO-MCP]` Campo `output_field` (padrão **`sql_result`**). No modo `query`, os dados; nos
modos `execute`/`execute_many`, o efeito (`[LACUNA]` o contrato não declara o formato do relatório
de linhas afetadas — ver L38).

`[CONFIRMADO-MCP]` `output_schema` permite **declarar** o schema de saída `[{name, type}]` para
validação — útil quando o downstream precisa de contrato estável mesmo que a query mude.

## Erros comuns

`[CONFIRMADO-MCP]` Placeholder em **comentário** → bind obrigatório fantasma.

`[CONFIRMADO-MCP]` Parâmetro `None` → recusado **antes** de rodar o SQL.

`[CONFIRMADO-MCP]` `''` como "sem valor" em **Oracle** → vira `NULL`, filtro muda de sentido **em
silêncio**.

`[VÍDEO]` Parâmetro ligado a campo do upstream no modo **Query** → *"Estáb não encontrado"*. Use
`Exec Many`. — `m3p2b:117-122`

`[VÍDEO]` `connection_id` ausente → *"Connection string obrigatória"*. — `m3p2b:115`

`[CONFIRMADO-MCP]` `timeout_seconds` padrão de 60 s estourado em script longo.

`[CONFIRMADO-DOC]` **Não é exportável** para SQL/Python: `sql_script` está na lista de tipos que
devolvem **HTTP 422** na exportação ("I/O externa"). Só o round-trip YAML o preserva.
— `guias-de-uso/exportar-e-importar.md § Cobertura V1`

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// SELECT parametrizado por variável do fluxo
{"script": "SELECT * FROM tabela WHERE id = :id",
 "mode": "query",
 "parameters": {"id": {"mode": "variable", "variable": "ID"}}}

// UPDATE iterando sobre cada linha do upstream
{"script": "UPDATE pedidos SET status = 'processado' WHERE id = :pedido_id",
 "mode": "execute_many",
 "parameters": {"pedido_id": {"mode": "dynamic", "template": "{{pedido_id}}"}}}
```

`[VÍDEO]` Montado em aula:

```sql
-- modo Exec Many, parâmetro :estab ligado ao campo ESTAB do nó Manual
SELECT * FROM filial WHERE estab = :estab
```
— `m3p2b:111-122`

```sql
-- modo Execute: bloco anônimo chamando função nativa do ConstruShow
DECLARE ... BEGIN ... SEQ_PRIMARY_KEY ... END;
```
— `m3p2b:127-130` (o texto exato do bloco não aparece na transcrição — `[LACUNA]`)

## Observações para o piloto de margem

1. **É o nó que corrige o preço.** O segundo exemplo oficial do contrato
   (`UPDATE pedidos SET status = … WHERE id = :pedido_id`, `execute_many`) é literalmente a forma
   do passo de correção do piloto, trocando `status` por preço. `[CONFIRMADO-MCP]`
2. **`unknown_write` significa que ninguém vai te avisar.** Diferente de `bulk_insert`, a
   plataforma não consegue classificar o script como escrita antes de rodar. Qualquer guardrail —
   dry-run, teto de variação, disjuntor — tem de ser **construído no fluxo**, não esperado do motor.
3. **Dry-run tem uma forma natural aqui:** o **mesmo script** no modo `query` (ou um `SELECT` que
   projete o valor novo ao lado do antigo) em vez de `execute_many`. `[INFERIDO]` — a plataforma
   não tem primitiva de dry-run (L2), e trocar `mode` é a aproximação mais próxima. **Risco:** é
   uma troca manual de um campo, sem trava; nada impede publicar com `execute_many` por engano.
4. **`''` = `NULL` no Oracle é o bug de margem esperando acontecer.** Um filtro
   `WHERE tabela_preco = :tabela` com `:tabela` vazio não filtra "todas" — não casa nada. O piloto
   tem de **remover o trecho do SQL** quando o filtro é opcional.
5. **Comparar com `bulk_insert` antes de decidir.** Para atualizar preço em volume:
   `sql_script`/`execute_many` faz **um round-trip por linha** (`[INFERIDO]` a partir da semântica
   "itera sobre upstream"), enquanto `bulk_insert`/`upsert` faz por lote e tem `on_update` por
   coluna — a peça achada para idempotência (L6). Contra: `bulk_insert` não sabe fazer `UPDATE`
   condicional complexo.
6. **`timeout_seconds` = 60 é apertado demais para o piloto.** Um `execute_many` sobre alguns
   milhares de pedidos precisa de valor explícito. `[LACUNA]` O que acontece com as linhas já
   aplicadas quando o timeout corta no meio (L37) — sem isso, não há como afirmar idempotência.
7. **Não é exportável.** O piloto não terá versão SQL/Python auditável do passo de correção de
   preço — só o YAML do fluxo. `[CONFIRMADO-DOC]`
