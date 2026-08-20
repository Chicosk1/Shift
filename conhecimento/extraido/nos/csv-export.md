# Nó `csv_export` — Exportar CSV

| | |
|---|---|
| **Tipo (MCP)** | `csv_export` |
| **Rótulo na interface** | Exportar CSV `[UI-OBSERVADA]` — `m3p4:60` |
| **Categoria** | `output` (grupo *Saídas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Exporta o dataset upstream para um **arquivo CSV no servidor**, disponível para
download via URL. Suporta configuração de delimitador, encoding e **BOM UTF-8 para Excel**.
— `describe_node('csv_export')`

`[VÍDEO]` Em aula: *"basicamente ele vai gerar um arquivo CSV dos dados que você passar pra cá"*.
— `m3p4:61`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('csv_export')`
- Gerar um arquivo CSV para download após o processamento.
- Exportar dados para integração com sistemas externos **via arquivo**.
- Criar relatórios em formato tabular.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('csv_export')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição | Rótulo na tela |
|---|---|---|---|---|---|
| `filename` | string | não | `export.csv` | Nome do arquivo | "nome" `[UI-OBSERVADA]` `m3p4:62-63` |
| `delimiter` | string | não | `,` | Separador de colunas | "delimitador" `[UI-OBSERVADA]` `m3p4:63` |
| `include_header` | boolean | não | `true` | Inclui linha de cabeçalho | "incluir cabeçalhos" `[UI-OBSERVADA]` `m3p4:64` |
| `encoding` | enum | não | *(não declarado)* | `utf-8`, `utf-16`, `latin-1`, `iso-8859-1`, `cp1252`, `ascii` | `[LACUNA]` não observado na tela |
| `quote_all` | boolean | não | *(não declarado)* | Se `true`, **todos** os campos são citados | "os campos de texto você quer adicionar aspas em todos" `[UI-OBSERVADA]` `m3p4:64` |

`[VÍDEO]` A aula preencheu `filename` = `funcionarios` e manteve o delimitador em vírgula:
*"aqui você define qual que é o delimitador que você quer. Por padrão geralmente é vírgula."*
— `m3p4:62-63`

> `[LACUNA]` **O MCP declara "BOM UTF-8 para Excel" na descrição do nó, mas não expõe parâmetro
> algum para controlá-lo** na lista de configuração. — `describe_node('csv_export')`. Falta
> descobrir: é automático quando `encoding: utf-8`? É um campo booleano oculto do contrato? Sem BOM,
> acento abre torto no Excel — é o problema mais comum de CSV entregue a usuário de negócio.

> `[LACUNA]` **Sem valor padrão declarado** para `encoding` e `quote_all` no contrato do MCP (os
> outros três campos declaram padrão explícito). `[INFERIDO]` `utf-8` e `false`, por simetria com o
> `csv_input` e com `include_index` do `xlsx_export`. Falta confirmar.

`[LACUNA]` **Não há guia de uso para este nó.** Existe `guias-de-uso/nos/csv.md`, mas ele documenta
somente o `csv_input` (*"Tipo interno: `csv_input`"*, § frontmatter). Todas as afirmações acima vêm
do MCP e da aula. Falta descobrir: limite de linhas (o `xlsx_export` declara 1.048.576; o
`csv_export` **não declara nenhum**), destino do arquivo no servidor, e se há política de retentativa.

## Entradas esperadas

`[CONFIRMADO-MCP]` Um dataset upstream — "o dataset upstream". — `describe_node('csv_export')`

`[VÍDEO]` Em aula o upstream foi `internal_data_source` (base interna de funcionários, 301 linhas).
— `m3p4:17-19`, `m3p4:106`

`[LACUNA]` **Não está documentado o que acontece com 0 linhas de entrada** — gera arquivo só com
cabeçalho, gera arquivo vazio, ou falha? O `csv_input` declara que arquivo vazio **falha**
(`guias-de-uso/nos/csv.md § Comportamentos importantes`), mas isso é o lado da leitura. Impacto
direto na observabilidade: ver `procedimentos/monitorar-execucao.md`.

## Saídas produzidas

`[CONFIRMADO-MCP]` Um arquivo no servidor, **disponível para download via URL**.
— `describe_node('csv_export')`

> 🔥 `[VÍDEO]` **Onde baixar o arquivo — a pegadinha central desta aula.** O botão de download no
> painel de resultado do nó **baixa só o preview**, e *"o preview é só 500 linhas"*. O arquivo
> completo fica em **Execuções → (última execução) → arquivos gerados → Baixar**.
>
> > *"Mas como que eu baixo? Não é nesse botão aqui. Esse botão é pra baixar o preview. Porque se
> > você tiver, o preview é só 500 linhas, aqui tem só 301 nessa base né, mas o preview é 500 linhas
> > no máximo. Se for um arquivo que tenha sei lá 2000 linhas o local correto de você baixar é aqui
> > em execuções... Daí nessa última execução ele vai ter aqui ó: arquivos gerados. E daí aqui sim é
> > o arquivo completo."* — `m3p4:68-75`
>
> Consequência prática: com uma base de 301 linhas (o caso da aula) os dois caminhos dão o mesmo
> resultado e **o erro passa despercebido**. Só acima de 500 linhas o preview trunca em silêncio.

`[LACUNA]` O `pdf_report` declara `output_mode` (`link`/`base64`/`file`) e faz **passthrough** do
dataset upstream para permitir encadeamento `[CONFIRMADO-MCP]`. O `csv_export` **não declara nem
`output_mode` nem passthrough**. Falta descobrir se o `csv_export` interrompe a cadeia (o que impede
"exportar CSV **e depois** notificar sobre ele no mesmo ramo) — porém o `gmail_send` aceita anexo por
`{source: 'node', node_id: '<id do nó que gerou o arquivo>'}` `[CONFIRMADO-MCP]`
(`describe_node('gmail_send')`), o que **sugere** que a referência do arquivo é alcançável por id de
nó sem precisar de passthrough. `[INFERIDO]`, falta confirmar.

## Erros comuns

> ⚠️ `[VÍDEO]` **Fluxo em produção exige publicar antes de executar.** Duas vezes na aula o
> instrutor executou, viu o resultado antigo e percebeu: *"ah eu esqueci de salvar né. Como tá em
> produção... Daí ele fica a versão anterior né."* — `m3p4:66-67`. E de novo no Excel: *"ah eu tenho
> que só salvar aqui né, porque ele tá em produção, publicar a versão."* — `m3p4:87`
>
> `[VÍDEO]` **Mas produção não é requisito do nó:** *"Ah por que que cê tá em produção? Funciona em
> teste também tá, só... não tem nenhuma... nenhuma limitação de eu estar em produção."* — `m3p4:89`

> 🔥 `[VÍDEO]` **Não há como enviar o arquivo gerado — segundo a aula.** *"até o momento eu não
> tenho opção de você enviar isso aqui né é... de alguma maneira né, mas a ideia é que também no
> futuro a gente venha numas atualizações poder enviar isso aqui por email, eh subir isso em algum
> lugar, enfim. Então por enquanto ele te dá a opção de você vir aqui e pegar esse arquivo
> manualmente."* — `m3p4:76-78`
>
> ⚠️ **DIVERGÊNCIA — o futuro já chegou.** `[CONFIRMADO-MCP]` O nó **`gmail_send` existe** e aceita
> anexo `{source: 'node', node_id: '<id do nó que gerou o arquivo>'}`, com o caso de uso declarado
> *"Mandar o relatório em PDF ou a planilha que o fluxo gerou"*. — `describe_node('gmail_send')`.
> **Registre as duas fontes:** a aula (`m3p4:76-78`) diz que não dá; o MCP diz que dá. A aula é
> anterior ao nó. Ver `procedimentos/monitorar-execucao.md` e a lacuna L23 no relatório do lote.

`[CONFIRMADO-MCP]` `encoding` é **enum fechado** — valor fora da lista deve ser rejeitado.
`[INFERIDO]` por simetria com `csv_input`, onde encoding fora da lista dá erro de validação
(`guias-de-uso/nos/csv.md § Limites e guardrails`).

`[LACUNA]` **Não está documentado se `delimiter` do `csv_export` exige exatamente 1 caractere.** O
`csv_input` exige `[CONFIRMADO-DOC]` (`guias-de-uso/nos/csv.md § Limites e guardrails`); o contrato
do `csv_export` só diz "string". Falta confirmar.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como exportar um CSV separado por ponto e vírgula?"*:

```json
{"filename": "relatorio.csv", "delimiter": ";", "include_header": true, "encoding": "utf-8"}
```

`[VÍDEO]` Em aula: fluxo *"Saídas e Lógica"* com `manual` → `internal_data_source` (base
`funcionarios`, 301 linhas) → `csv_export` com `filename` = `funcionarios`, delimitador vírgula,
cabeçalho incluído, sem aspas em todos. Publicado, executado, e o arquivo baixado em
**Execuções → arquivos gerados**. O CSV foi conferido abrindo no Excel e no Notepad++.
— `m3p4:59-94`
