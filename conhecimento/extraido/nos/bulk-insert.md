# Nó `bulk_insert` — Bulk Insert / Inserção em Massa

| | |
|---|---|
| **Tipo (MCP)** | `bulk_insert` |
| **Rótulo na interface** | Inserção em Massa |
| **Categoria** | `database` (grupo *Banco de Dados*) |
| **Risco** | **`write`** `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

> **Nó de escrita.** Toda a seção *Guardrails* de `plano-skills-shift.md` §5.4 existe por causa
> de nós como este. Não configurar sem antes ler `checklist-pre-producao`.

## O que faz

`[CONFIRMADO-MCP]` Insere linhas do dataset upstream em uma tabela de banco de dados **externo**,
com mapeamento de colunas. — `describe_node('bulk_insert')`

## Quando usar

`[CONFIRMADO-MCP]`
- Carregar dados de uma transformação em tabela de banco externo.
- Fazer **upsert** de registros usando chave de merge.
- Inserir **apenas registros novos** (`insert_if_not_exists`).
- Processar grandes volumes em lotes (`batch_size` configurável).

`[CONFIRMADO-DOC]` Alternativa sem banco externo: **Base de Dados Interna (Escrita)**
(`internal_data_write`), que grava no banco da própria plataforma. Serve para testar sem
conexão, mas é **para cadastros pequenos — teto de 200 mil linhas**. Para carga de produção
real, conexão externa continua sendo o caminho. — `primeiros-passos.md §4`

## ⚠️ Armadilha declarada no contrato

`[CONFIRMADO-MCP]` Citado literalmente, porque falha tarde:

> **Firebird NÃO é suportado para escrita**: o nó recusa a execução quando a conexão é Firebird.
> Se o destino do usuário for um Firebird, diga isso **ANTES** de montar — o nó fica pronto no
> canvas e a **falha só aparece na primeira execução**.

Relevante porque `conceitos.md § Conexão` lista Firebird entre os bancos suportados
`[CONFIRMADO-DOC]` — suportado para **leitura**, não para escrita. Não é conflito, é assimetria
que a doc não explicita.

> **Não aplicável a este projeto.** O escopo é **somente Oracle** (decisão de 2026-08-20). O fato
> fica registrado, mas não precisa virar alerta na skill.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('bulk_insert')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `connection_id` | string | **sim** | — | UUID do conector SQL de destino |
| `target_table` | string | **sim** | — | Tabela de destino (ex.: `schema.tabela`) |
| `column_mapping` | array | **sim** | — | `[{source, target}]` ou `[{value, target}]` |
| `load_strategy` | enum | não | `append_fast` | `append_fast` / `append_safe` / `upsert` / `insert_if_not_exists` |
| `merge_keys` | array | não | — | Colunas-chave para `upsert` / `insert_if_not_exists` |
| `batch_size` | number | não | `1000` | Tamanho do lote de insert |
| `returning_columns` | array | não | — | Colunas a retornar após insert (ex.: IDs auto-gerados) |
| `unique_columns` | array | não | — | Colunas para **deduplicação interna antes** do insert |
| `delivery_mode` | enum | não | `auto` | `auto` / `tunnel` / `edge` — como aplicar quando a conexão é via relay |

### Estratégias de carga

`[CONFIRMADO-DOC]` + `[CONFIRMADO-MCP]` — as 4 batem exatamente entre doc e MCP:

| Estratégia | Comportamento |
|---|---|
| `append_fast` | **Padrão.** Só insere |
| `append_safe` | Mesma coisa, mas **desfaz tudo se uma linha falhar** |
| `upsert` | Insere ou atualiza. **Exige `merge_keys`** |
| `insert_if_not_exists` | Insere só o que não existe. **Exige `merge_keys`** |

— `primeiros-passos.md §4`, `describe_node('bulk_insert')`

### `on_update` — controle fino por coluna no upsert

`[CONFIRMADO-MCP]` Cada entrada de `column_mapping` aceita `on_update`, que **só vale em
`upsert`**:

| Valor | Comportamento |
|---|---|
| `overwrite` | **Padrão.** Sobrescreve sempre |
| `keep_if_empty` | Valor vazio que chega **preserva** o que está gravado |
| `never` | Coluna entra no insert e **nunca é atualizada** |

Isto não aparece em nenhuma aula nem documento — é exclusivo do contrato do MCP.

### `delivery_mode` — quando a conexão passa por relay

`[CONFIRMADO-MCP]` `auto` usa borda quando elegível, senão túnel; `tunnel` força túnel; `edge`
força borda e **dá erro se inelegível**.

> `[CONFIRMADO-DOC]` Contexto que muda a leitura: `RELAY_APPLY_MIN_ROWS` é 10.000 e a flag
> `RELAY_APPLY_ROW_TOLERANCE_ENABLED` está **OFF por default**. Ou seja, o caminho de borda
> provavelmente **não está ativo** no ambiente. — `configuracao.md`, `extracao-na-borda-design.md`
> `[LACUNA]` Falta confirmar o valor real das flags neste ambiente.

## Entradas esperadas

Um dataset tabular upstream, com colunas compatíveis com `column_mapping`.
`[CONFIRMADO-DOC]` A tabela de destino precisa **já existir** no banco. — `primeiros-passos.md § O que você vai precisar`

## Saídas produzidas

`[CONFIRMADO-DOC]` `row_count` — quantas linhas foram gravadas, visível no painel de execução.
— `primeiros-passos.md §6`

`[CONFIRMADO-MCP]` Opcionalmente as `returning_columns`, úteis para capturar IDs auto-gerados.

## Erros comuns

`[CONFIRMADO-MCP]` Conexão Firebird → nó recusa a execução (ver armadilha acima).

`[CONFIRMADO-MCP]` `upsert` ou `insert_if_not_exists` sem `merge_keys` — as duas estratégias
exigem a chave. `[CONFIRMADO-DOC]` `primeiros-passos.md §4` reforça: *"essas duas últimas exigem
indicar qual coluna identifica um registro único"*.

`[CONFIRMADO-DOC]` Falha típica reportada pela mensagem de erro: coluna ausente, tipo
incompatível — a mensagem aponta o nó e o motivo. — `primeiros-passos.md §6`

`[CONFIRMADO-MCP]` Linhas rejeitadas vão para **dead-letter**, inspecionável com `ler_rejeicoes`
(grupos por causa, amostras com o campo culpado, campos sensíveis mascarados).

`[CONFIRMADO-DOC]` **Não é exportável** para SQL/Python: `bulk_insert` está na lista de tipos que
devolvem HTTP 422 na exportação. Só o round-trip YAML o preserva.
— `guias-de-uso/exportar-e-importar.md § Cobertura V1`

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Append simples
{"target_table": "schema.tabela", "column_mapping": [{"source": "col_a", "target": "col_a"}], "load_strategy": "append_fast"}

// Upsert por id
{"target_table": "clientes", "column_mapping": [{"source": "id", "target": "id"}, {"source": "nome", "target": "nome"}], "load_strategy": "upsert", "merge_keys": ["id"]}

// Atualizar cadastro sem apagar campos que não vieram
{"target_table": "clientes", "column_mapping": [
   {"source": "id", "target": "id"},
   {"source": "numero", "target": "numero", "on_update": "keep_if_empty"},
   {"source": "forma_pagamento", "target": "forma_pagamento", "on_update": "never"}],
 "load_strategy": "upsert", "merge_keys": ["id"]}
```

`[VÍDEO]` Em aula: nó renomeado para **"Insere Tabela Funcionários"**. Conexão salva
pré-existente, tabela `Funcionarios`, estratégia **Insert** — o instrutor menciona que poderia
usar **upsert** *"pra ele verificar se já existe"*, mas escolheu insert por rapidez. Usou
**auto-mapear**, que reaproveitou o schema já conhecido da execução anterior. Resultado: 300
funcionários gravados em ~2 segundos, conferidos com `SELECT *`. A tabela foi criada à mão antes,
com `CREATE TABLE` no gerenciador de banco. — `m1:71-76`

## Observações para o piloto de margem

1. **`on_update: 'never'` e `keep_if_empty` são a peça mais relevante achada até aqui para
   idempotência** (L6). Permitem que reprocessar o mesmo pedido não sobrescreva campos que o
   fluxo não deveria tocar, sem depender de lógica condicional no fluxo.
2. **`returning_columns`** permite capturar o valor gravado — insumo para a trilha de auditoria
   pedida em §5 do plano.
3. **`unique_columns`** deduplica **antes** do insert, o que ataca a duplicação criada de
   propósito pela margem de sobreposição da janela incremental (§5.1).
4. `append_safe` é a única estratégia com semântica transacional ("desfaz tudo se uma linha
   falhar") — candidata natural para escrita de preço, em vez do `append_fast` padrão.
5. **Destino é Oracle** — sem a restrição de escrita do Firebird. Em contrapartida, vale
   verificar o bug conhecido de **Oracle + dlt** registrado em
   `planejamento-dlt-(extracao-insercao).md`, que joga Oracle em fallback SQLAlchemy e pode
   afetar desempenho em volume.
