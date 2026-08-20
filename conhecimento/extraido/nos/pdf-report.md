# Nó `pdf_report` — Relatório PDF

| | |
|---|---|
| **Tipo (MCP)** | `pdf_report` |
| **Rótulo na interface** | Relatório PDF `[CONFIRMADO-MCP]` — `describe_node('pdf_report')`. `[LACUNA]` não observado na tela |
| **Categoria** | `output` (grupo *Saídas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `relatorio_pdf`, `report_pdf`, `pdf` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Gera um documento PDF rico (título, indicadores/KPIs, gráficos de
barras/linha/pizza e tabelas) **vinculando colunas já prontas** dos resultados dos nós anteriores.
Retorna uma URL de download e **faz passthrough do dataset upstream para permitir encadeamento**.
— `describe_node('pdf_report')`

> 🔥 `[CONFIRMADO-MCP]` **O nó NÃO agrega dados.** *"qualquer soma/média/agrupamento deve ser feita
> num nó anterior (ex.: SQL)."* — `describe_node('pdf_report')`
>
> Ele apenas *vincula* colunas que já existem. Um bloco de gráfico com `value_column: "total"` exige
> que uma coluna chamada `total` já esteja no dataset — o nó não a calcula. O caminho é
> `aggregator` (Group By) ou `sql_script` antes.

`[CONFIRMADO-MCP]` **É o único nó de saída que declara passthrough**, o que o torna encadeável: o
dataset upstream continua disponível para os nós seguintes. O `csv_export` e o `xlsx_export` não
declaram isso. — comparação entre `describe_node('pdf_report')`, `describe_node('csv_export')` e
`describe_node('xlsx_export')`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('pdf_report')`
- Entregar um relatório apresentável (PDF) para um gestor ou cliente.
- Resumir os dados processados em KPIs, gráficos e tabelas num documento.
- Gerar um PDF com indicadores e gráficos ao final de um fluxo.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('pdf_report')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `filename` | string | não | `relatorio.pdf` | Nome do arquivo PDF |
| `output_mode` | string | não | `link` | `link` (download no detalhe da execução), `base64` (PDF em base64 no output/resposta de webhook), `file` (referência de arquivo para o próximo nó) |
| `meta` | objeto | não | — | `title`, `subtitle`, `logo`, `accent_color`, `page_size` (`a4`\|`letter`), `orientation` (`portrait`\|`landscape`) |
| `variables` | objeto | não | — | Valores para substituir `{{variavel}}` nos blocos de texto |
| `template_id` | string | não | — | Id de um **Modelo de Relatório do espaço de trabalho** |
| `blocks` | array | **sim, quando não há `template_id`** | — | Lista de blocos: `text`, `kpi_row`, `chart`, `table` |

### `blocks` — a estrutura do documento

`[CONFIRMADO-MCP]` Cada bloco vincula colunas já prontas de um nó de origem (`source`).
— `describe_node('pdf_report')`

| Tipo de bloco | Campos de vínculo |
|---|---|
| `text` | usa `variables` para substituir `{{variavel}}` |
| `kpi_row` | `column` (**primeiro registro**) ou `value` fixo |
| `chart` | `label_column` / `value_column`; `chart_type` (`bar`, `line`, `pie`); `limit` |
| `table` | `columns[]` |

> ⚠️ `[CONFIRMADO-MCP]` **`kpi_row` com `column` lê o PRIMEIRO REGISTRO da coluna, não a soma.**
> — `describe_node('pdf_report')`. É a consequência prática de "o nó não agrega": para um KPI
> "faturamento total", o nó anterior tem de devolver **uma linha** com o total já calculado (é
> exatamente o caso do `aggregator` com `group_by` vazio, que *"agrega todas as linhas em um único
> registro"* `[CONFIRMADO-MCP]` — `list_nodes()`).

### `template_id` — estrutura no espaço de trabalho

> ⚠️ `[CONFIRMADO-MCP]` **`template_id` e `blocks` são mutuamente exclusivos, com precedência do
> template:** *"Com ele, a ESTRUTURA do documento vem do modelo (**mudar o modelo muda todos os
> fluxos que o usam**) e `blocks` é **ignorado**. Sem ele, o documento é o definido em `blocks`
> aqui."* — `describe_node('pdf_report')`
>
> Duas armadilhas: (1) editar `blocks` com um `template_id` preenchido **não tem efeito nenhum**, sem
> aviso; (2) o Modelo de Relatório é um recurso **compartilhado do espaço de trabalho** — alterá-lo
> muda relatórios de outros fluxos.

`[LACUNA]` **"Modelo de Relatório" é uma entidade não documentada em lugar nenhum do material bruto**
— só aparece nesta frase do contrato do MCP. Falta descobrir: onde se cria, quem pode editar, se há
menu próprio na interface, e se o `template_id` é listável por alguma ferramenta do MCP (não há
`list_report_templates` no catálogo de ferramentas).

## Entradas esperadas

`[CONFIRMADO-MCP]` Um ou mais datasets upstream, referenciados por bloco através do campo `source`.
**As colunas de agregação têm de vir prontas.** — `describe_node('pdf_report')`

`[LACUNA]` **Não está documentado o formato do `source`** — id do nó, rótulo do nó, ou nome do campo
de saída? O exemplo do contrato omite `source` inteiramente no bloco `chart`, sugerindo um default
(o único upstream?). Falta descobrir.

## Saídas produzidas

`[CONFIRMADO-MCP]` Depende de `output_mode` — `describe_node('pdf_report')`:
- `link` (padrão): download no **detalhe da execução** (a mesma tela "arquivos gerados" descrita em
  `m3p4:73` `[VÍDEO]`).
- `base64`: o PDF vai em base64 no output / **na resposta do webhook** — este é o modo para devolver
  o PDF a quem chamou o fluxo por API.
- `file`: referência de arquivo **para o próximo nó**.

`[CONFIRMADO-MCP]` Além disso, **passthrough do dataset upstream**, então os nós seguintes continuam
vendo os dados originais. — `describe_node('pdf_report')`

## Erros comuns

> 🔥 `[CONFIRMADO-MCP]` **Esperar que o nó agregue.** É a armadilha declarada no próprio contrato:
> soma/média/agrupamento **tem de vir de nó anterior**. Um `chart` com `value_column: "total"` sobre
> um dataset de linhas cruas plota linha a linha, não o total.
> — `describe_node('pdf_report')`

`[CONFIRMADO-MCP]` **`blocks` obrigatório quando não há `template_id`** — sem nenhum dos dois, não há
documento a gerar. — `describe_node('pdf_report')`

`[CONFIRMADO-MCP]` **`blocks` silenciosamente ignorado** quando `template_id` está preenchido.
— `describe_node('pdf_report')`

> ⚠️ **DIVERGÊNCIA — o nó não existe na aula.** `[VÍDEO]` A aula de saídas do módulo 3 enumera as
> saídas e fecha com *"com isso a gente fechou aqui as opções que a gente tem até o momento de nós de
> saída"* — `m3p4:160` — sem mencionar PDF em nenhum momento (`m3p4:59-160` cobre só Saída do Fluxo,
> CSV, Excel e Google Sheets). `[CONFIRMADO-MCP]` O `pdf_report` está no catálogo de hoje.
> — `list_nodes(category='output')`. **A aula é anterior ao nó.** Consequência: nenhum rótulo de tela
> deste nó é observado, e não existe demonstração de uso real.

`[LACUNA]` **Não há guia de uso** para o `pdf_report` em `guias-de-uso/nos/`. Todo o conteúdo deste
arquivo vem de uma única fonte (`describe_node('pdf_report')`). Falta descobrir: limite de páginas,
limite de linhas em bloco `table`, tamanho máximo do PDF, o que acontece com `logo` (URL? upload?),
e se `accent_color` aceita nome de cor ou só hex.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Gerar um PDF com o total de vendas por filial (já agregado
por um nó SQL anterior) em um gráfico de barras."* Note o parêntese: **o contrato já assume a
agregação a montante**.

```json
{"filename": "vendas.pdf",
 "meta": {"title": "Vendas por filial", "accent_color": "#0f9d6e"},
 "blocks": [{"type": "chart", "chart_type": "bar", "title": "Total por filial",
             "label_column": "filial", "value_column": "total", "limit": 10}]}
```

`[LACUNA]` **Nenhum exemplo de `kpi_row`, `table`, `text` ou `variables` existe no material.** Falta
descobrir o esquema completo de cada tipo de bloco — os nomes de campo listados acima
(`label_column`, `value_column`, `columns[]`, `column`, `value`, `source`) vêm da descrição em prosa
do contrato, não de um esquema formal.
