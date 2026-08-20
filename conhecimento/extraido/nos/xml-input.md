# Nó `xml_input` — Entrada XML

| | |
|---|---|
| **Tipo (MCP)** | `xml_input` |
| **Rótulo na interface** | Entrada XML |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases aceitos** | `xml`, `nfe`, `nota fiscal xml`, `arquivo xml` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Lê um arquivo XML e devolve **uma linha por ocorrência do elemento escolhido
pelo usuário** (ex.: `/pedidos/pedido`). Elementos filhos e atributos viram colunas;
sub-elementos repetidos (os itens de um pedido) viram **coluna de lista** que o nó `unnest`
(*Desaninhar*) expande. **Documento com namespace é lido sem declarar prefixo.** Leitura **em
fluxo**: arquivo grande não é carregado inteiro em memória. — `describe_node('xml_input')`

## Quando usar

`[CONFIRMADO-MCP]`
- Importar XML de integração ou de sistema legado para o fluxo.
- Ler cadastro, pedido ou retorno de serviço que chegou em XML.
- Transformar um XML enviado pela tela em dados tabulares.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('xml_input')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `url` | string | **sim** | — | Caminho local ou URL do arquivo XML. Aceita `shift-upload://<file_id>` |
| `record_path` | string | **sim** | — | Elemento repetido que vira LINHA |
| `encoding` | string | não | `auto` | `auto` respeita o prólogo do documento. Lista fechada: `auto`, `utf-8`, `iso-8859-1`, `cp1252`, `utf-16` |
| `attribute_prefix` | string | não | `@` | Prefixo das colunas vindas de ATRIBUTO |
| `list_elements` | array | não | — | Nomes de sub-elemento que saem SEMPRE como lista, mesmo aparecendo uma vez só (ex.: `['item']`) |
| `max_rows` | number | não | — | Limite de linhas a ler |
| `output_field` | string | não | `data` | Nome do campo de saída |

### `record_path` não é XPath

> `[CONFIRMADO-MCP]` *"Elemento repetido que vira LINHA: '/pedidos/pedido' (a partir da raiz) ou
> só 'pedido' (em qualquer profundidade). **Não é XPath**: só nomes de elemento separados por '/',
> sem predicado."* — `describe_node('xml_input')`

Ou seja: caminho com `/` inicial ancora na raiz; nome nu casa em qualquer profundidade. Predicado
(`[@id='1']`), curinga, eixo e função de XPath **não** são suportados.

### `attribute_prefix` — quando usar vazio

> `[CONFIRMADO-MCP]` *"Prefixo das colunas que vêm de ATRIBUTO (padrão: '@'). Use '' só se não
> houver atributo e elemento com o mesmo nome."* — `describe_node('xml_input')`

## Entradas esperadas

Nenhuma tabular — é nó de entrada. `[LACUNA]` O contrato não declara qual nó antecede.

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma tabela no campo `output_field` (padrão `data`):
- **uma linha por ocorrência** do elemento em `record_path`;
- um **elemento filho** → uma coluna;
- um **atributo** → uma coluna com o prefixo de `attribute_prefix` (padrão `@id`);
- um **sub-elemento repetido** → uma **coluna de lista**, expansível pelo `unnest`;
- **toda coluna é TEXTO.**

## Erros comuns

> ⚠️ `[CONFIRMADO-MCP]` **Tudo vira texto:** *"TODA coluna sai como TEXTO — XML não tem tipo.
> Número, data e valor monetário chegam como string ('10.50', '2026-08-19'). Encadeie o nó Mapper
> para converter ANTES de somar, comparar ou gravar em coluna numérica. Em troca, código com zero
> à esquerda (CEP, CNPJ, código de barras) chega intacto."* — `describe_node('xml_input')`

> ⚠️ `[CONFIRMADO-MCP]` **Atributo exige aspas em SQL:** *"Atributo vira coluna com o prefixo '@'
> ('@id'), e isso EXIGE aspas em SQL escrito à mão (\"@id\"). Se o fluxo tiver um nó de SQL
> depois, considere renomear no Mapper."* — `describe_node('xml_input')`

> ⚠️ `[CONFIRMADO-MCP]` **A janela de 500 registros:** *"Sub-elemento repetido vira coluna de
> LISTA, que só se expande pelo nó Desaninhar (array_field aceita caminho: 'itens.item'). Se o
> mesmo sub-elemento aparece 1× num registro e 3× em outro DEPOIS dos primeiros 500 registros,
> declare o nome em 'list_elements' — senão a coluna degrada para JSON e o Desaninhar recusa."*
> — `describe_node('xml_input')`

Esta é a armadilha mais custosa das três: o sintoma só aparece com arquivo grande e a cardinalidade
tem de ser prevista *antes* de rodar. A defesa é declarar o nome em `list_elements` sempre que
houver dúvida — o custo de declarar sem precisar é nenhum (a coluna vira lista de um elemento, e o
`unnest` a expande normalmente).

`[LACUNA]` O contrato não declara o que acontece com XML malformado: `xml_input` **não tem
parâmetro `on_error`** (o `xml_to_rows`, seu par, tem). Se o arquivo é inválido, não está dito se
o nó derruba ou pula.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como ler os pedidos de um arquivo XML?"*:

```json
{"url": "shift-upload://<file_id>", "record_path": "/pedidos/pedido", "encoding": "auto"}
```

## Observações

`[CONFIRMADO-MCP]` **Par de nós com o mesmo parser.** `xml_input` lê um **arquivo**;
`xml_to_rows` (categoria `transform`) lê uma **coluna que contém XML** — e o contrato deste diz
*"Mesmo parse do nó de Entrada XML (namespace, atributos, sub-elementos repetidos)"*. As três
armadilhas de tipagem/atributo/lista valem para os dois. Para resposta de API não-JSON, o nó certo
é o `xml_to_rows`, não este.

`[CONFIRMADO-MCP]` O alias `nfe` / `nota fiscal xml` indica NF-e como caso de uso previsto, mas o
contrato **não traz layout, `record_path` nem exemplo de NF-e** — só o alias. `[LACUNA]`

`[LACUNA]` **`.xml` não está na lista de extensões aceitas no upload** (`.csv .tsv .xlsx .xls
.json .parquet .txt`, fixas e não configuráveis). O parâmetro `url` aceita URL, o que resolve o
caso remoto, mas como um `.xml` chega à Área de Arquivos não está declarado.
