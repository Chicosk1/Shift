# Nó `loadNode` — Carga SQL (Load) / "Destino SQL"

| | |
|---|---|
| **Tipo (MCP)** | `loadNode` |
| **Aliases (MCP)** | `load_node`, **`destino_sql`** |
| **Rótulo no contrato do MCP** | Carga SQL (Load) |
| **Rótulo na interface (aula)** | **Destino SQL** — `m3p2b:97,171` `[UI-OBSERVADA]` |
| **Categoria** | `database` (grupo *Banco de Dados*) `[CONFIRMADO-MCP]` |
| **Risco** | **`write`** `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | ⚠️ **Em disputa** — a aula diz `bulk_insert`; o MCP não marca nada. Ver abaixo |

> **Nó de escrita.** Não configurar sem antes ler `checklist-pre-producao`.

## ⚠️ DIVERGÊNCIA CENTRAL — este nó está descontinuado?

**Registro das duas fontes, sem escolher lado.**

**Fonte A — aula `[VÍDEO]`** `m3p2b:170`, sobre o nó **Destino SQL**:

> *"Esse aqui eu acho que ele nem tá mais… mais para ser usado. Deixa eu só confirmar aqui. (…)
> Esse aqui era, era uma opção a mais que a gente tinha. Eu acho que até tá funcionando, mas **ele
> vai ser descontinuado**, né? **O INSERT já englobou o que ele faz.**"*

Note que a própria fala é hedged (*"eu acho que"*, *"deixa eu só confirmar"*, futuro *"vai ser"*) —
é opinião do instrutor, não anúncio.

**Fonte B — MCP `[CONFIRMADO-MCP]`** `describe_node('loadNode')` e `list_nodes(category='database')`:
o nó **aparece normalmente** no catálogo, com contrato completo, três estratégias de escrita e
**nenhuma marca de deprecação** — nem no campo de descrição, nem em "Armadilhas conhecidas"
(que este nó não tem). Compare com o padrão que a base já registrou para nós consolidados: o MCP
sinaliza sucessão quando existe (ex.: `join` → *"Versão legada — prefira o nó 'combine' com
mode='join'"*, `describe_node`/`list_nodes`). **`loadNode` não traz nada disso.**

**Fonte C — documentação `[CONFIRMADO-DOC]`** `guias-de-uso/exportar-e-importar.md § Cobertura V1`
lista `loadNode` como o **único nó de saída exportável** para SQL/Python ("Saída: `loadNode`
(gerado como comentário `-- TODO: write to …`)"), enquanto `bulk_insert` está entre os que devolvem
**HTTP 422**. Um nó em vias de sumir dificilmente seria o único suportado no exportador.

`[INFERIDO]` As três fontes juntas sugerem que a deprecação é **intenção do time, não estado da
plataforma**. Mas isso é dedução. Ver `divergencias.md` e L39.

### A afirmação "o INSERT já englobou o que ele faz" é falsa num ponto

`[CONFIRMADO-MCP]` Comparando os contratos: `bulk_insert` tem `append_fast`, `append_safe`,
`upsert` e `insert_if_not_exists` — **não tem `replace`**. O `write_disposition: replace`
(TRUNCATE + INSERT) do `loadNode` **não tem equivalente direto** em `bulk_insert`; o caminho
equivalente é `truncate_table` + `bulk_insert`, ou seja **dois nós**. Registrado como contraponto
factual à fala de `m3p2b:170`.

## O que faz

`[CONFIRMADO-MCP]` Carrega o dataset upstream para uma tabela em banco de dados **externo**.
Suporta `append`, `replace` (**TRUNCATE + INSERT**) e `merge` (**UPSERT por chave**).
— `describe_node('loadNode')`

## Quando usar

`[CONFIRMADO-MCP]`
- Carregar dados processados para uma tabela de destino em banco externo.
- Atualizar tabela de destino por chave (`merge`/upsert).
- **Substituir completamente** a tabela de destino (`replace`).

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('loadNode')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `connection_id` | string | **sim** | — | UUID do conector SQL de destino |
| `target_table` | string | **sim** | — | Tabela de destino (pode incluir schema, ex.: `schema.tabela`) |
| `write_disposition` | enum | não | `append` | `append` / `replace` / `merge` |
| `merge_keys` | array | não | — | Colunas-chave para `merge`/upsert |
| `chunk_size` | number | não | `1000` | Tamanho do lote de streaming |
| `output_field` | string | não | `load_result` | Nome do campo de saída |

### `write_disposition`

`[CONFIRMADO-MCP]`

| Valor | Comportamento |
|---|---|
| `append` | **Padrão.** Só acrescenta |
| `replace` | **TRUNCATE + INSERT** — apaga tudo antes de gravar |
| `merge` | UPSERT por chave. **Exige `merge_keys`** `[INFERIDO]` (o contrato não marca a dependência como obrigatória, mas o parâmetro só existe para isso) |

> ⚠️ `replace` **destrói o conteúdo da tabela de destino** antes de inserir. Não há campo de
> confirmação e não há `where_clause` como no `truncate_table` — é tudo ou nada.

### O que este nó **não** tem (e o `bulk_insert` tem)

`[CONFIRMADO-MCP]` Comparação de contratos, relevante para escolher entre os dois:

| Recurso | `loadNode` | `bulk_insert` |
|---|---|---|
| Mapeamento explícito de colunas (`column_mapping`) | **não** | sim |
| `on_update` por coluna (`overwrite`/`keep_if_empty`/`never`) | **não** | sim |
| `unique_columns` (dedup antes do insert) | **não** | sim |
| `returning_columns` | **não** | sim |
| `delivery_mode` (borda vs. túnel) | **não** | sim |
| Semântica transacional explícita (`append_safe`) | **não** | sim |
| `replace` (TRUNCATE + INSERT em um nó) | **sim** | **não** |
| Exportável para SQL/Python | **sim** `[CONFIRMADO-DOC]` | não (HTTP 422) |
| `chunk_size` padrão | 1.000 | 1.000 |

`[INFERIDO]` Sem `column_mapping`, o `loadNode` casa colunas **por nome** entre o dataset upstream
e a tabela de destino. O contrato não declara isso — é dedução a partir da ausência do parâmetro.
Ver L40.

## Entradas esperadas

Um dataset tabular upstream. `[INFERIDO]` Com colunas cujos nomes correspondam às da tabela de
destino, que precisa **já existir** (mesma condição que `primeiros-passos.md § O que você vai
precisar` declara `[CONFIRMADO-DOC]` para o `bulk_insert`; o contrato do `loadNode` não repete).

## Saídas produzidas

`[CONFIRMADO-MCP]` Campo `output_field`, padrão **`load_result`**. `[LACUNA]` O contrato não
declara o conteúdo do relatório (linhas gravadas, rejeitadas, duplicatas removidas). Ver L38.

## Erros comuns

`[LACUNA]` **O contrato deste nó não declara nenhuma armadilha** — é o único nó de escrita do lote
sem seção "Armadilhas conhecidas". Não se sabe se isso significa "não tem" ou "não foi
documentado". Concretamente, ficam abertas:
- Vale a **restrição de Firebird para escrita** que o `bulk_insert` declara? *(irrelevante no
  escopo Oracle, mas é a mesma família de nó)*
- O `merge` do `loadNode` passa pelo mesmo `load_service` do `bulk_insert`, e portanto pelo mesmo
  gargalo de **1 MERGE por linha** descrito em
  `upsert-rapido-staging-+-merge-set-based-(design).md §1`? `[INFERIDO]` provavelmente sim (o
  documento fala de `load_service` genericamente, não de um nó), mas **não confirmado**. Ver L41.

`[CONFIRMADO-DOC]` **É exportável**, com ressalva: o exportador gera a escrita como
**comentário** `-- TODO: write to …`, ou seja o SQL exportado **não escreve**. O `SELECT *` final
imprime o resultado do nó upstream do `loadNode`.
— `guias-de-uso/exportar-e-importar.md § Cobertura V1`, `§ Rodando o SQL exportado`

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Merge pelo cnpj
{"target_table": "clientes_processed",
 "write_disposition": "merge",
 "merge_keys": ["cnpj"],
 "connection_id": "<uuid>"}
```

`[VÍDEO]` **Não há demonstração em aula.** O nó foi inserido no canvas apenas para ser mostrado e
imediatamente removido: *"Exibição visual rápida dos painéis e propriedades do nó 'Destino SQL'
inserido provisoriamente"* / *"O fluxo é limpo para remover nós não utilizados"* — `m3p2b:171-172`.
Nenhum parâmetro foi preenchido na tela.

## Observações para o piloto de margem

1. **Não use este nó no piloto sem antes fechar a divergência de deprecação (L39).** Construir o
   passo de escrita sobre um nó que o time considera "a caminho da descontinuação" é dívida
   garantida — mesmo que hoje funcione.
2. **`bulk_insert` é o substituto certo** para o piloto: tem `on_update` por coluna (a peça de
   idempotência, L6), `unique_columns` (absorve a duplicata criada pelo `reprocess_window`) e
   `returning_columns` (insumo de auditoria). O `loadNode` não tem nenhum dos três.
3. **A única razão para preferir `loadNode` é a exportabilidade** `[CONFIRMADO-DOC]`: é o único nó
   de saída que sobrevive à exportação para SQL/Python. Se o piloto precisar de artefato auditável
   fora da plataforma, isso pesa — mas a escrita sai como comentário, então o valor é de
   documentação, não de execução.
4. **`replace` é armadilha num fluxo agendado.** Combinado com o teto de 500 linhas do modo Teste
   (`m2p5:105`), um `replace` rodado em Teste **apaga a tabela inteira e regrava 500 linhas**.
   `[INFERIDO]` — a combinação não foi testada, mas decorre das duas propriedades confirmadas
   isoladamente. Ver L42.
