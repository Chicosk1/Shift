# Nó `ofx_input` — Entrada OFX (Extrato)

| | |
|---|---|
| **Tipo (MCP)** | `ofx_input` |
| **Rótulo na interface** | Entrada OFX (Extrato) |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases aceitos** | `extrato`, `extrato bancario`, `ofx` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Lê um extrato bancário no formato OFX (*"o que todo banco exporta"*) e devolve
**uma linha por transação**, com FITID, data, valor assinado, tipo, memo e número do documento.
**Dados da conta e saldo ficam no resultado do nó** — não nas linhas. Aceita **OFX 1.x (SGML) e
2.x (XML)**. — `describe_node('ofx_input')`

## Quando usar

`[CONFIRMADO-MCP]`
- Importar extrato bancário para conciliação.
- Trazer lançamentos do banco para dentro de um fluxo.
- Ler o extrato que o cliente exportou do internet banking.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('ofx_input')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `url` | string | **sim** | — | Caminho local do arquivo OFX. Aceita `shift-upload://<file_id>` |
| `encoding` | string | não | `auto` | `auto` segue o que o arquivo declara. Lista fechada: `auto`, `utf-8`, `iso-8859-1`, `cp1252` |
| `max_rows` | number | não | — | Limite de transações a ler |
| `output_field` | string | não | `data` | Nome do campo de saída |

`[LACUNA]` O contrato não descreve as cinco origens de arquivo da interface para este nó (só
declara que `url` aceita `shift-upload://`). Também não diz se há parâmetro de retentativa nem se
a leitura é em fluxo — diferente do `xml_input`, onde a leitura em fluxo é declarada
explicitamente.

## Entradas esperadas

Nenhuma tabular — é nó de entrada. `[LACUNA]` O contrato não declara qual nó antecede.

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma tabela no campo `output_field` (padrão `data`), com **uma linha por
transação**. Colunas declaradas no contrato: **FITID, data, valor assinado, tipo, memo e número do
documento**.

`[CONFIRMADO-MCP]` **Dados da conta e saldo ficam no resultado do nó**, fora da tabela de linhas.

`[LACUNA]` O contrato nomeia os campos em português corrido (`FITID`, `data`, `valor`, `tipo`,
`memo`, `número do documento`) mas só grafa literalmente o nome técnico de um deles — `fitid`, na
armadilha. Os nomes exatos das outras colunas e a estrutura de "dados da conta e saldo" no
resultado do nó não estão declarados.

## Erros comuns

> ⚠️ `[CONFIRMADO-MCP]` **Sinal do valor:** *"O valor vem ASSINADO: débito é negativo, crédito
> positivo. Não aplique valor absoluto nem inverta sinal por conta própria — a conciliação depende
> disso."* — `describe_node('ofx_input')`

> ⚠️ `[CONFIRMADO-MCP]` **FITID repete entre execuções:** *"O identificador único do lançamento é
> o `fitid`, e ele repete entre execuções (o mesmo extrato baixado duas vezes traz os mesmos
> FITIDs). Para não reprocessar, encadeie o nó de deduplicação por `fitid` — este nó não
> deduplica."* — `describe_node('ofx_input')`

`[CONFIRMADO-MCP]` O nó de deduplicação citado é o `deduplication` (*Deduplicar*, categoria
`transform`) — `list_nodes(query='duplicad')`.

`[LACUNA]` O contrato não diz o que acontece com arquivo OFX malformado: não há parâmetro
`on_error` nem menção a linha pulada, ao contrário do `xml_to_rows`. Também não declara limite de
tamanho próprio.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como ler o extrato OFX que o cliente enviou?"*:

```json
{"url": "shift-upload://<file_id>", "encoding": "auto"}
```

## Observações

`[LACUNA]` **`.ofx` não está na lista de extensões aceitas no upload.** As extensões permitidas
são fixas e não configuráveis: `.csv .tsv .xlsx .xls .json .parquet .txt`. O contrato do nó diz
que `url` aceita `shift-upload://<file_id>`, mas não explica como um `.ofx` chega à Área de
Arquivos. O `google_drive_download` — que grava o arquivo como ativo do fluxo e devolve a mesma
referência `shift-upload://<id>` — é um caminho plausível, e o exemplo do `google_drive_list`
filtra justamente por `.ofx`; mas **o contrato não declara que o download do Drive escapa da lista
de extensões**. Isso precisa ser verificado antes de prometer o cenário ao usuário.

`[CONFIRMADO-MCP]` Combinação declarada com os nós do Drive: `google_drive_list` trata o filtro por
extensão `.ofx` como *"o jeito certo de pegar só os .ofx"*, e o exemplo do
`google_drive_download` é *"Como baixar cada extrato que a listagem do Drive achou?"*. Os três nós
foram desenhados para o mesmo cenário de conciliação bancária.
