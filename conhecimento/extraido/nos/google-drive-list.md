# Nó `google_drive_list` — Google Drive (Listar arquivos)

| | |
|---|---|
| **Tipo (MCP)** | `google_drive_list` |
| **Rótulo na interface** | Google Drive (Listar arquivos) |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases aceitos** | `[LACUNA]` — o contrato não declara alias |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Lista os arquivos de uma pasta do Google Drive **como dataset do fluxo** — uma
linha por arquivo, com **id, nome, tipo, tamanho, extensão e data de modificação**. **Pagina a
pasta inteira** e aceita filtro por extensão exata (*"o jeito certo de pegar 'só os .ofx'"*), por
trecho do nome e por data de modificação. — `describe_node('google_drive_list')`

O que o nó entrega é **metadado**, não conteúdo: nenhum arquivo é baixado aqui. O `id` de cada
linha é a matéria-prima do `google_drive_download`.

## Quando usar

`[CONFIRMADO-MCP]`
- Descobrir o que chegou numa pasta compartilhada na nuvem.
- Processar arquivos que um parceiro deposita numa pasta do Drive.
- Rodar de tempos em tempos e agir só sobre o que é novo.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('google_drive_list')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `connection_id` | string | **sim** | — | ID da conexão (credencial OAuth) Google do usuário |
| `folder_id` | string | **sim** | — | **ID ou URL** da pasta do Drive (`root` = raiz do Meu Drive) |
| `extension` | string | não | vazio / `all` = todos | Extensão exigida, **com ponto** (`.ofx`, `.xml`, `.csv`) |
| `name_contains` | string | não | — | Só arquivos cujo nome contém este trecho (ex.: `extrato`) |
| `mime_types` | string | não | — | Avançado. Tipos MIME separados por vírgula |
| `modified_after` | string | não | — | Só arquivos modificados depois desta data (`AAAA-MM-DD` ou ISO 8601) |
| `include_folders` | boolean | não | `false` | Incluir subpastas na listagem |
| `max_files` | number | não | — | Limite total de arquivos listados, **contado DEPOIS do filtro de extensão** |
| `output_field` | string | não | `data` | Nome do campo de saída |

### Os três filtros e a hierarquia entre eles

Este nó tem três formas de filtrar por identidade do arquivo, e o contrato **declara qual é a
certa** e por que as outras duas falham:

> `[CONFIRMADO-MCP]` **`extension`** — *"Extensão exigida, com ponto ('.ofx', '.xml', '.csv');
> vazio ou 'all' = todos os arquivos. É o filtro correto para 'só os .ofx' — confere o sufixo
> exato, sem diferenciar maiúsculas."*

> ⚠️ `[CONFIRMADO-MCP]` **`name_contains`** — *"Só arquivos cujo nome contém este trecho (ex.:
> 'extrato'). É SUBSTRING: não use para extensão — '.ofx' aqui casaria 'extrato.ofx.bak'."*

> ⚠️ `[CONFIRMADO-MCP]` **`mime_types`** — *"Avançado. Tipos MIME separados por vírgula (ex.:
> 'application/pdf'). Não confie nele para OFX/XML: o MIME vem de quem subiu o arquivo — prefira
> 'extension'."*

Regra prática que o contrato estabelece: **filtro de tipo de arquivo = `extension`, sempre.**
`name_contains` é para o *nome* (o trecho `extrato`), `mime_types` é escape hatch avançado com
confiabilidade declaradamente ruim para os formatos de interesse.

### Interações declaradas entre parâmetros

> `[CONFIRMADO-MCP]` **`include_folders`** — *"Incluir subpastas na listagem (padrão: false).
> **Ignorado quando há extensão exigida** — pasta não tem extensão."*

> `[CONFIRMADO-MCP]` **`max_files`** — *"Limite total de arquivos listados, contado **DEPOIS** do
> filtro de extensão."*

A ordem do `max_files` é a informação útil: `extension: '.ofx'` + `max_files: 10` devolve **10
arquivos `.ofx`**, não os 10 primeiros arquivos da pasta dos quais alguns são `.ofx`.

## Entradas esperadas

Nenhuma tabular — é nó de entrada. Depende de uma **conexão OAuth Google** já criada
(`connection_id`).

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma tabela no campo `output_field` (padrão `data`), **uma linha por arquivo**,
com: **id, nome, tipo, tamanho, extensão e data de modificação**.

`[CONFIRMADO-MCP]` **Pagina a pasta inteira** — o nó não devolve só a primeira página da API do
Drive.

`[LACUNA]` Os **nomes exatos das colunas** não estão declarados. O contrato do
`google_drive_download` usa `{{item.id}}` no exemplo, o que confirma **`id`** como nome da coluna
do identificador; os outros cinco (nome, tipo, tamanho, extensão, data de modificação) só aparecem
descritos em português corrido, sem grafia técnica.

`[LACUNA]` Não está declarado o que a coluna "tipo" contém — MIME type, ou uma classificação
própria do Shift.

## Erros comuns

`[CONFIRMADO-MCP]` As duas armadilhas deste nó estão declaradas **dentro das descrições dos
parâmetros**, não em seção separada, e ambas são sobre escolher o filtro errado:

> ⚠️ `[CONFIRMADO-MCP]` *"É SUBSTRING: não use para extensão — '.ofx' aqui casaria
> 'extrato.ofx.bak'."* — sobre `name_contains`

> ⚠️ `[CONFIRMADO-MCP]` *"Não confie nele para OFX/XML: o MIME vem de quem subiu o arquivo —
> prefira 'extension'."* — sobre `mime_types`

`[LACUNA]` **"Rodar de tempos em tempos e agir só sobre o que é novo"** é caso de uso declarado, e
`modified_after` é o parâmetro que o serve — mas o contrato **não declara como o valor é
mantido entre execuções**. Não há menção a marca d'água, estado persistido ou valor "última
execução": ou o `modified_after` recebe template/variável (não declarado), ou o incremental tem de
ser montado à mão. Sem isso, "só o que é novo" não fecha.

`[LACUNA]` Nada declarado sobre: **pasta vazia** (tabela de zero linhas, ou erro), **erro de
permissão** na pasta, **token OAuth expirado**, **teto de arquivos** que a paginação suporta, ou
**profundidade** — se `include_folders: true` lista as subpastas como linhas ou desce
recursivamente dentro delas.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como pegar os extratos OFX que chegam numa pasta do
Drive?"*:

```json
{
  "connection_id": "<uuid>",
  "folder_id": "https://drive.google.com/drive/folders/<ID>",
  "extension": ".ofx",
  "modified_after": "2026-08-01"
}
```

O exemplo usa a **URL** da pasta em `folder_id`, não o id nu — confirmando que colar o link do
navegador funciona.

## Observações

`[CONFIRMADO-MCP]` **Primeiro nó do trio de processamento de pasta.** A listagem produz o dataset
com `id` por arquivo; o `loop` (categoria `logic`, *"Itera sobre cada linha de um dataset invocando
um sub-workflow ou corpo inline por item"*) itera as linhas; o `google_drive_download` resolve
`{{item.id}}` e baixa um arquivo por volta, devolvendo `shift-upload://<id>`; um nó de entrada de
arquivo lê o conteúdo. Os três contratos apontam uns para os outros.

`[CONFIRMADO-MCP]` **Este nó é seguro de rodar para inspeção.** É `read_only` e não baixa nada —
serve para conferir *o que existe* numa pasta antes de montar o processamento, e o custo de errar o
filtro é ver linha errada, não gastar cota de download.

`[LACUNA]` O contrato não diz se `folder_id` aceita template/variável. `google_drive_download`
declara explicitamente que `file_id` *"aceita template"*; a ausência da mesma frase aqui não prova
o contrário, mas impede afirmar que a pasta pode ser escolhida em tempo de execução.
