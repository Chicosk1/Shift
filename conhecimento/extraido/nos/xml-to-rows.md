# Nó `xml_to_rows` — XML em Linhas

| | |
|---|---|
| **Tipo (MCP)** | `xml_to_rows` |
| **Rótulo na interface** | XML em Linhas |
| **Categoria** | `transform` `[CONFIRMADO-MCP]` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases aceitos** | `xml`, `parse xml`, `resposta xml`, `xml de campo` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Transforma **uma coluna que CONTÉM XML** em dados tabulares: cada ocorrência do
elemento escolhido vira uma linha, e **as demais colunas da entrada são carregadas em cada linha
gerada**. É o nó para tratar **resposta de API que devolve XML** — o corpo de um nó HTTP não-JSON
chega na coluna `value` e é consumido direto, **sem salvar arquivo**. *"Mesmo parse do nó de
Entrada XML (namespace, atributos, sub-elementos repetidos)."*
— `describe_node('xml_to_rows')`

A distinção com o `xml_input` é a fonte do XML, não o parser: `xml_input` lê **arquivo**,
`xml_to_rows` lê **coluna**.

## Quando usar

`[CONFIRMADO-MCP]`
- Tratar a resposta de uma API que devolve XML em vez de JSON.
- Explodir um campo de banco que guarda XML em linhas.
- Ler retorno de serviço fiscal ou de órgão público dentro do fluxo.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('xml_to_rows')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `xml_column` | string | **sim** | — | Coluna da entrada que contém o XML. **Para a resposta de um nó HTTP não-JSON, é `value`** |
| `record_path` | string | **sim** | — | Elemento repetido que vira LINHA |
| `parent_columns` | array | não | todas, exceto a do XML | Colunas da entrada a carregar em cada linha gerada |
| `child_prefix` | string | não | — | Prefixo para as colunas vindas do XML (ex.: `xml_`) — resolve colisão com coluna homônima da entrada |
| `attribute_prefix` | string | não | `@` | Prefixo das colunas que vêm de ATRIBUTO do XML |
| `list_elements` | array | não | — | Nomes de sub-elemento que saem SEMPRE como lista, mesmo aparecendo uma vez só |
| `on_error` | string | não | `skip` | Linha de campo vazio ou XML inválido: `skip` (pula e segue) ou `fail` (derruba o nó) |
| `max_output_rows` | number | não | — | Limite de linhas geradas |
| `output_field` | string | não | `data` | Nome do campo de saída |

### `record_path` não é XPath

> `[CONFIRMADO-MCP]` *"Elemento repetido que vira LINHA: '/pedidos/pedido' (a partir da raiz) ou
> só 'pedido' (em qualquer profundidade). **Não é XPath**: só nomes de elemento separados por
> '/'."* — `describe_node('xml_to_rows')`

## Entradas esperadas

`[CONFIRMADO-MCP]` Uma tabela com **uma coluna que contém XML como texto**. O caso canônico
declarado é a saída de um nó HTTP cuja resposta não é JSON: o corpo chega na coluna **`value`**.

`[LACUNA]` O contrato não declara se há teto de tamanho para o XML dentro de uma célula, nem se a
leitura do conteúdo da coluna é em fluxo (o `xml_input` declara leitura em fluxo para arquivo;
aqui não há declaração equivalente).

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma tabela no campo `output_field` (padrão `data`) em que cada linha combina:
- as colunas **do pai** (todas as da entrada exceto a do XML, ou o subconjunto de
  `parent_columns`);
- as colunas **vindas do XML** (opcionalmente prefixadas por `child_prefix`), uma linha por
  ocorrência de `record_path`;
- **toda coluna vinda do XML é TEXTO.**

`[CONFIRMADO-MCP]` Com `on_error='skip'`, o total de linhas descartadas aparece em
**`output_summary.rows_skipped`**.

## Erros comuns

> ⚠️ `[CONFIRMADO-MCP]` **Tudo vira texto:** *"TODA coluna vinda do XML sai como TEXTO — XML não
> tem tipo. Converta número e data no nó Mapper antes de somar ou gravar em coluna numérica. Em
> troca, código com zero à esquerda (CEP, CNPJ) chega intacto."* — `describe_node('xml_to_rows')`

> ⚠️ `[CONFIRMADO-MCP]` **O `skip` silencioso — a armadilha central deste nó:** *"Com
> on_error='skip' (o padrão) linha ruim é PULADA em silêncio: o total aparece em
> output_summary.rows_skipped, não como erro. Se cada linha importa (retorno fiscal, por exemplo),
> use on_error='fail' — senão uma resposta vazia do destino passa batida."*
> — `describe_node('xml_to_rows')`

> ⚠️ `[CONFIRMADO-MCP]` **Colisão de nome derruba o nó mesmo com `skip`:** *"Colisão de nome entre
> coluna da entrada e campo do XML FALHA o nó, mesmo com on_error='skip' — é problema de
> configuração, não de dado. Resolva com 'child_prefix' ou escolhendo 'parent_columns'."*
> — `describe_node('xml_to_rows')`

A leitura combinada das duas últimas: `on_error` cobre **problema de dado**, não **problema de
configuração**. O padrão `skip` esconde resposta vazia e derruba o nó em colisão de nome — os dois
comportamentos são o inverso do que a intuição sugere.

`[LACUNA]` O contrato não declara o que conta como "XML inválido" para o `on_error` — se cobre
só XML não-parseável, ou também XML válido em que `record_path` não casa nenhum elemento.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"A API respondeu XML; como transformo em linhas?"*:

```json
{"xml_column": "value", "record_path": "/pedidos/pedido", "on_error": "skip"}
```

Note que o exemplo do contrato usa `skip`, o padrão — mas a própria armadilha declarada recomenda
`fail` quando cada linha importa. Para retorno fiscal, siga a armadilha, não o exemplo.

## Observações

`[CONFIRMADO-MCP]` **Este é o nó certo para API que devolve XML.** O contrato é explícito:
*"É o nó para tratar resposta de API que devolve XML — o corpo de um nó HTTP não-JSON chega na
coluna 'value' e é consumido direto, sem salvar arquivo."* Não há passo intermediário de gravar
arquivo e reler com `xml_input`.

`[CONFIRMADO-MCP]` Herda do `xml_input` a expansão de sub-elemento repetido: a coluna de lista se
expande pelo nó `unnest` (*Desaninhar*), cujo `array_field` aceita caminho (`itens.item`).

`[LACUNA]` O `xml_input` declara explicitamente a **janela de 500 registros** que determina se uma
coluna vira lista ou degrada para JSON. O contrato do `xml_to_rows` diz *"mesmo parse"* e oferece o
mesmo `list_elements`, mas **não repete o número 500** nem diz se a janela é por célula, por linha
de entrada ou pelo conjunto. Se o comportamento importa, precisa ser verificado.
