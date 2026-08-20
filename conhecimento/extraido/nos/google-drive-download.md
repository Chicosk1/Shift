# Nó `google_drive_download` — Google Drive (Baixar arquivo)

| | |
|---|---|
| **Tipo (MCP)** | `google_drive_download` |
| **Rótulo na interface** | Google Drive (Baixar arquivo) |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases aceitos** | `[LACUNA]` — o contrato não declara alias |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Baixa um arquivo do Google Drive e **o guarda como ativo do fluxo**, devolvendo
a referência **`shift-upload://<id>`** que os nós de entrada de arquivo (CSV, Excel) leem. *"O id
do arquivo aceita valor ligado ao passo anterior, para baixar um por iteração dentro de um nó
Loop."* — `describe_node('google_drive_download')`

A peça-chave é a referência de saída: este nó não entrega **dados**, entrega o **endereço de um
arquivo** no formato que os nós de entrada já sabem consumir. É uma ponte, não uma leitura.

## Quando usar

`[CONFIRMADO-MCP]`
- Trazer para o Shift um arquivo que está numa pasta do Drive.
- Processar, um a um, os arquivos que a listagem do Drive achou.
- Ler um extrato ou planilha de terceiro sem baixar na mão.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('google_drive_download')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `connection_id` | string | **sim** | — | ID da conexão (credencial OAuth) Google do usuário |
| `file_id` | string | **sim** | — | ID do arquivo no Drive. **Aceita template**, ex.: `{{item.id}}` dentro de um Loop |
| `file_name` | string | não | o nome que está no Drive | Nome a gravar na Área de Arquivos |
| `folder` | string | não | — | Pasta de destino na Área de Arquivos (ex.: `/Extratos`) |
| `max_size_mb` | number | não | — | Teto de tamanho em MB; **acima disso o arquivo é recusado antes de baixar** |
| `output_field` | string | não | `data` | Nome do campo de saída |

### `file_id` aceita template — é o que habilita o loop

`[CONFIRMADO-MCP]` A sintaxe declarada é `{{item.id}}`, e o contexto é *"dentro de um Loop"*. É
por isso que o padrão listar → iterar → baixar funciona: o `google_drive_list` produz uma linha por
arquivo com o `id`, o `loop` itera as linhas, e cada iteração resolve `{{item.id}}` para o arquivo
daquela volta.

### `max_size_mb` recusa antes de baixar

`[CONFIRMADO-MCP]` O contrato é explícito quanto ao momento: *"acima disso o arquivo é recusado
**antes** de baixar"*. É proteção de banda e de cota, não validação pós-download.
`[LACUNA]` O que o nó faz ao recusar — derrubar o fluxo, ou pular a iteração — **não está
declarado**. Não há parâmetro `on_error` neste nó. Numa pasta com um arquivo grande no meio, esse é
o comportamento que decide se o lote inteiro cai.

## Entradas esperadas

`[CONFIRMADO-MCP]` Nenhuma obrigatória — é nó de entrada. Mas o caso de uso principal declarado é
receber a ligação de um passo anterior, para que `file_id` resolva `{{item.id}}` por iteração.
Funciona nos dois modos: `file_id` fixo (um arquivo conhecido) ou por template (dentro de `loop`).

## Saídas produzidas

`[CONFIRMADO-MCP]` No campo `output_field` (padrão `data`), a referência **`shift-upload://<id>`**
do arquivo gravado como ativo do fluxo — a mesma forma que `csv_input`, `ofx_input`, `xml_input` e
o nó de Excel aceitam em `url`.

`[LACUNA]` A **estrutura exata da saída** não está declarada: se é uma tabela de uma linha, quais
colunas traz além da referência (nome gravado, tamanho, pasta de destino), e qual o nome do campo
que carrega o `shift-upload://`. Para ligar este nó a um `csv_input` é justamente esse nome que
precisa ser referenciado.

## Erros comuns

`[LACUNA]` **O contrato não declara armadilha para este nó.** O que não está dito e importa:

`[LACUNA]` **Interação com a lista fixa de extensões do upload.** A Área de Arquivos aceita apenas
`.csv .tsv .xlsx .xls .json .parquet .txt`, fixas e não configuráveis. Este nó grava o arquivo
como ativo do fluxo — e o exemplo do `google_drive_list` filtra `.ofx`, extensão que **não está na
lista**. O contrato não diz se o download do Drive está sujeito à mesma lista. É a lacuna mais
consequente do nó: se estiver sujeito, o cenário de conciliação OFX via Drive não fecha.

`[LACUNA]` **Interação com os limites da Área de Arquivos** (500 MB por arquivo, 5 GB por projeto,
TTL de 30 dias). O nó tem seu próprio `max_size_mb`, mas o contrato não diz como os dois tetos se
relacionam, nem se o arquivo baixado num loop de 200 arquivos conta 200 vezes contra os 5 GB do
projeto — e se conta, o que acontece quando a cota estoura no meio.

`[LACUNA]` **Nada declarado sobre a conexão OAuth:** o que ocorre com token expirado ou revogado,
nem se há mensagem distinta de "arquivo não encontrado" versus "sem permissão".

`[LACUNA]` **Google Docs / Sheets nativos.** Arquivo nativo do Google não é um arquivo binário com
extensão — precisa ser exportado. O contrato não menciona conversão, formato de exportação, nem o
que acontece ao tentar baixar um Google Sheets por este nó.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como baixar cada extrato que a listagem do Drive
achou?"*:

```json
{"connection_id": "<uuid>", "file_id": "{{item.id}}", "folder": "/Extratos"}
```

A pergunta do exemplo já pressupõe o padrão de três nós: a listagem achou, o loop itera, este nó
baixa.

## Observações

`[CONFIRMADO-MCP]` **Faz par com `google_drive_list`.** A listagem produz `id` por arquivo; este nó
consome esse `id`. Os dois contratos se referenciam mutuamente — a listagem se descreve como
*"Processar arquivos que um parceiro deposita numa pasta do Drive"*, e este nó como *"Processar, um
a um, os arquivos que a listagem do Drive achou"*.

`[CONFIRMADO-MCP]` **Não é um nó de leitura de dados.** Nada é tabulado aqui: o arquivo continua
arquivo. Sempre exige um nó de entrada depois (`csv_input`, `ofx_input`, `xml_input`, Excel)
apontando `url` para a referência devolvida. Um fluxo que baixa e não encadeia leitura não produz
dado nenhum.

`[CONFIRMADO-MCP]` `folder` sugere organização por destino na Área de Arquivos (`/Extratos` no
exemplo). `[LACUNA]` O contrato não diz se a pasta é criada quando não existe, nem o que ocorre
quando dois arquivos com o mesmo `file_name` caem na mesma pasta — sobrescreve, renomeia ou falha.
Num loop sobre uma pasta com nomes repetidos, isso importa.
