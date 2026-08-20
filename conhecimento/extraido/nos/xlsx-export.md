# Nó `xlsx_export` — Exportar XLSX

| | |
|---|---|
| **Tipo (MCP)** | `xlsx_export` |
| **Rótulo na interface** | Exportar Excel `[UI-OBSERVADA]` — `m3p4:80` |
| **Categoria** | `output` (grupo *Saídas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Exporta o dataset upstream para um **arquivo Excel (`.xlsx`) gerado no servidor**.
Retorna URL de download e metadados do arquivo. **Limitado a 1.048.576 linhas (limite do Excel).**
— `describe_node('xlsx_export')`

`[VÍDEO]` Em aula, apresentado como gêmeo do CSV: *"Então aqui dentro da opção de saída, a gente tem
também pra gerar uma planilha. Então é a mesma lógica do CSV"* e, no fim, *"a parte de CSV aqui e...
e planilha é basicamente a mesma lógica, só muda o tipo do arquivo né."* — `m3p4:81`, `m3p4:98`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('xlsx_export')`
- Gerar um arquivo Excel a partir dos dados processados no fluxo.
- Entregar um relatório `.xlsx` para download automático.
- Exportar resultados para **revisão manual em planilha**.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('xlsx_export')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição | Rótulo na tela |
|---|---|---|---|---|---|
| `filename` | string | não | `export.xlsx` | Nome do arquivo a gerar | "nome" `[UI-OBSERVADA]` `m3p4:83` |
| `sheet_name` | string | não | `Dados` | Nome da aba. **Máx. 31 caracteres** | "nome da aba" `[UI-OBSERVADA]` `m3p4:84` |
| `include_index` | boolean | não | `false` | Se `true`, inclui coluna de índice | "Incluir coluna de índice" `[UI-OBSERVADA]` `m3p4:85-86` |
| `max_rows` | number | não | **`1.048.576`** | Limite de linhas a exportar (limite do Excel) | "quantidade máximo de linhas" `[UI-OBSERVADA]` `m3p4:84` |

`[VÍDEO]` A aula preencheu `filename` = `funcionarios_planilha` e `sheet_name` = `dados` (o padrão),
deixou `max_rows` no máximo (*"pode deixar sempre máximo aqui"*) e **desmarcou** `include_index`.
— `m3p4:83-87`

`[VÍDEO]` Explicação do `include_index` dada em aula: *"se você vai querer incluir índice, o que que
é isso? Ele vai gerar uma coluna com 1, 2, 3, 4, sabe. Então se você não quiser isso pode deixar
desmarcado."* — `m3p4:85-86`

> 🔥 `[CONFIRMADO-MCP]` **Limite de 1.048.576 linhas** — é o teto de linhas de uma planilha do Excel,
> não uma escolha do Shift. `max_rows` já vem nesse valor por padrão.
> — `describe_node('xlsx_export')`. **Falta descobrir** `[LACUNA]` o que acontece ao estourar: o nó
> falha, ou trunca em silêncio? `max_rows` funcionando como "limite" sugere truncamento silencioso, o
> que seria grave — uma exportação de preços incompleta e sem aviso.

`[LACUNA]` **Não há guia de uso para este nó** em `guias-de-uso/nos/`. Existe
`guias-de-uso/nos/excel.md`, mas ele documenta somente o `excel_input` (*"Tipo interno:
`excel_input`"*, § frontmatter). Todo o conteúdo acima vem do MCP e da aula. Falta descobrir:
formatação de célula (o `.xlsx` sai com tipos ou tudo texto?), largura de coluna, congelamento de
cabeçalho, e se há política de retentativa.

`[LACUNA]` **`sheet_name` com mais de 31 caracteres** — o contrato declara o limite `[CONFIRMADO-MCP]`
mas não o comportamento na violação (erro de validação ou truncamento). Falta descobrir.

`[LACUNA]` **Não há parâmetro de múltiplas abas.** Uma execução do nó gera uma aba. `[INFERIDO]` Por
simetria com o `excel_input` (*"Múltiplas abas → crie um nó Excel separado para cada aba"*,
`guias-de-uso/nos/excel.md § Comportamentos importantes`), o provável é um nó por aba — **mas isso
gera arquivos separados, não abas do mesmo arquivo.** Falta descobrir se dá para escrever N abas em
um único `.xlsx`.

## Entradas esperadas

`[CONFIRMADO-MCP]` Um dataset upstream. — `describe_node('xlsx_export')`

`[VÍDEO]` Em aula o upstream foi `internal_data_source` (base `funcionarios`, 301 linhas), o mesmo
usado pelo `csv_export` no mesmo fluxo. — `m3p4:17-19`, `m3p4:106`

`[LACUNA]` Mesma lacuna do `csv_export`: **não está documentado o que acontece com 0 linhas de
entrada.** Impacto na observabilidade — ver `procedimentos/monitorar-execucao.md`.

## Saídas produzidas

`[CONFIRMADO-MCP]` **URL de download e metadados do arquivo.** — `describe_node('xlsx_export')`

`[VÍDEO]` Na aba **Execuções**, o arquivo aparece na lista de arquivos gerados: *"Então aqui em
execuções da mesma forma ó, ele gerou aqui planilhas xlsx."* Os dois arquivos do fluxo (o CSV e o
XLSX) apareceram juntos na mesma execução. — `m3p4:90-92`

> 🔥 `[VÍDEO]` **Mesma pegadinha do CSV: o botão do painel baixa só o preview (500 linhas).** O
> arquivo completo está em **Execuções → arquivos gerados → Baixar**. A aula estabelece a regra no
> `csv_export` (`m3p4:68-75`) e depois declara *"O mesmo vale tá... pra... o mesmo vale aqui pro
> Excel."* — `m3p4:79`

`[LACUNA]` O `pdf_report` declara **passthrough** do dataset upstream para permitir encadeamento
`[CONFIRMADO-MCP]`; o `xlsx_export` **não declara passthrough**. Falta descobrir se o ramo termina
aqui. (O `gmail_send` aceita anexo por `node_id` do nó que gerou o arquivo `[CONFIRMADO-MCP]`, o que
sugere que o arquivo é alcançável mesmo sem passthrough — `describe_node('gmail_send')`.)

## Erros comuns

> ⚠️ `[VÍDEO]` **Fluxo em produção exige publicar a versão antes de executar**, senão você executa a
> versão anterior e vê resultado velho: *"ah eu tenho que só salvar aqui né, porque ele tá em
> produção, publicar a versão."* — `m3p4:87`
>
> `[VÍDEO]` Produção **não** é requisito do nó: *"Funciona em teste também tá, só... não tem
> nenhuma... nenhuma limitação de eu estar em produção."* — `m3p4:89`

`[CONFIRMADO-MCP]` **Não há parâmetro de `encoding`** neste nó (diferente do `csv_export`) — o
`.xlsx` é OOXML, o encoding é interno ao formato. Não é lacuna, é ausência correta.

> `[VÍDEO]` **Nó desativado não executa e não gera arquivo.** Mais adiante na mesma aula, o
> instrutor *desativou* os nós de exportar CSV e Excel para testar o Google Sheets sem gerar arquivo
> — `m3p4:112`. Ou seja: a ausência de um arquivo em "arquivos gerados" pode significar **nó
> desativado**, não falha. Ver `procedimentos/monitorar-execucao.md`.

> ⚠️ **DIVERGÊNCIA — o catálogo de saídas da aula está incompleto.** `[VÍDEO]` A aula fecha
> declarando *"com isso a gente fechou aqui as opções que a gente tem até o momento de nós de saída"*
> — `m3p4:160` — depois de mostrar apenas quatro: Saída do Fluxo (`workflow_output`), Exportar CSV,
> Exportar Excel e Gravar Google Sheets. `[CONFIRMADO-MCP]` Hoje existem ao menos **três saídas
> adicionais**: `pdf_report`, `gmail_send` e `zapi_send_text`. — `list_nodes(category='output')` e
> `describe_node`. **A aula é anterior a esses nós.**

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como exportar o resultado para um arquivo Excel chamado
'relatorio_mensal.xlsx'?"*:

```json
{"filename": "relatorio_mensal.xlsx", "sheet_name": "Dados"}
```

`[VÍDEO]` Em aula, no fluxo *"Saídas e Lógica"*: `manual` → `internal_data_source` (base
`funcionarios`) → `xlsx_export` com `filename` = `funcionarios_planilha`, `sheet_name` = `dados`,
`max_rows` no máximo, `include_index` desmarcado. Publicado, executado, e a planilha baixada em
**Execuções → arquivos gerados** e aberta no Excel: *"Então ele gera uma planilha com os dados."*
— `m3p4:80-97`
