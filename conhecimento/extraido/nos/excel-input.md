# Nó `excel_input` — Entrada Excel

| | |
|---|---|
| **Tipo (MCP)** | `excel_input` |
| **Rótulo na interface** | Excel `[CONFIRMADO-DOC]` — `guias-de-uso/nos/excel.md § title` / "Entrada Excel" no MCP `[CONFIRMADO-MCP]` |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Lê uma planilha Excel (`.xlsx`) local ou remota e **materializa o conteúdo em
DuckDB via streaming linha a linha**, sem carregar tudo em memória. — `describe_node('excel_input')`

`[CONFIRMADO-DOC]` O motor é o `openpyxl` em modo somente-leitura. Arquivos remotos são baixados
automaticamente antes da leitura. — `guias-de-uso/nos/excel.md § Descrição`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('excel_input')`
- Importar dados de planilha Excel para o fluxo.
- Ler arquivo Excel enviado pelo usuário via upload.
- Consumir relatórios Excel de URLs remotas.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('excel_input')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `url` | string | **sim** | — | Caminho local ou URL. Aceita `shift-upload://<file_id>` |
| `sheet_name` | string | não | — | **Nome** da aba. Ver divergência abaixo |
| `header_row` | number | não | `0` | Índice (base 0) da linha de cabeçalho |
| `skip_empty` | boolean | não | `true` | Ignora linhas completamente vazias |
| `max_rows` | number | não | — | Limite de linhas de dados a ler (não conta cabeçalho) |
| `output_field` | string | não | `data` | Nome do campo de saída |

`[CONFIRMADO-DOC]` O guia declara **dois parâmetros que o contrato do MCP não lista**:
`input_model_id` (UUID do Modelo de Entrada, para validar o cabeçalho) e `retry_policy` (objeto).
— `guias-de-uso/nos/excel.md § Configurações`

> ⚠️ **DIVERGÊNCIA — tipo de `sheet_name`.** Duas fontes se contradizem e nenhuma foi eleita:
> - `[CONFIRMADO-DOC]` "`sheet_name` | string ou inteiro | `null` | Nome da aba **ou índice
>   (base 0)**. `null` = aba ativa (primeira)". — `guias-de-uso/nos/excel.md § Configurações`
> - `[CONFIRMADO-MCP]` "`sheet_name` (string): **NOME da aba (texto)**. Para usar a primeira aba,
>   **OMITA** este campo — **não mande índice numérico**." — `describe_node('excel_input')`
>
> O guia ainda descreve o comportamento de índice fora do intervalo (§ Limites e guardrails), o que
> sugere que o backend aceita inteiro; o MCP proíbe explicitamente. **Falta descobrir** qual vale no
> runtime atual. Na dúvida, use o nome da aba — é o único aceito pelas duas fontes.

### Política de retentativa

`[CONFIRMADO-DOC]` Mesmos campos do nó CSV: `max_attempts` (1–10), `backoff_strategy`
(`none`/`fixed`/`exponential`), `backoff_seconds` (0.1–300), `retry_on` (lista de strings que filtra
retentativa por mensagem de erro). — `guias-de-uso/nos/excel.md § Política de retentativa` +
`guias-de-uso/nos/csv.md § Política de retentativa`

### Fontes de arquivo suportadas

`[CONFIRMADO-DOC]` — `guias-de-uso/nos/excel.md § Fontes de arquivo suportadas`

| Fonte | Exemplo | Quando usar |
|---|---|---|
| URL HTTP/HTTPS | `https://exemplo.com/relatorio.xlsx` | Arquivo público ou com auth na URL |
| Caminho local | `/data/relatorios/vendas.xlsx` | Arquivo no servidor Shift |
| Upload do projeto | picker | Arquivo enviado pela interface |
| Variável | `{{vars.ArquivoExcel}}` | Planilha que muda a cada execução |

> ⚠️ `[CONFIRMADO-DOC]` **S3/GCS/Azure não são suportados diretamente neste nó** — use URL pública
> ou caminho local após montar o storage. **Isto é uma assimetria em relação ao `csv_input`**, que
> carrega a extensão `httpfs` do DuckDB automaticamente e aceita `s3://` direto.
> — `guias-de-uso/nos/excel.md § Fontes de arquivo suportadas` vs. `guias-de-uso/nos/csv.md §
> Comportamentos importantes`

### Seleção de aba

`[CONFIRMADO-DOC]` — `guias-de-uso/nos/excel.md § Seleção de aba`
- **Sem modelo vinculado:** o picker lista as abas detectadas automaticamente no arquivo; também é
  possível digitar nome ou número.
- **Com modelo vinculado:** o picker lista **apenas as abas definidas no modelo**, e seleciona a
  primeira automaticamente se `sheet_name` estiver vazio.
- **Se a aba solicitada não existir, o nó usa a primeira aba disponível e registra um aviso no log
  — sem erro.** Pegadinha grave: erro de digitação no nome da aba não falha, lê a aba errada.

### Linha de cabeçalho (`header_row`)

`[CONFIRMADO-DOC]` Define qual linha contém os nomes das colunas; linhas anteriores são descartadas.
Se a linha de cabeçalho estiver vazia, as colunas recebem nomes automáticos `col_0`, `col_1`, etc.
— `guias-de-uso/nos/excel.md § Linha de cabeçalho`

> `[INFERIDO]` O `csv_input` sem cabeçalho nomeia as colunas `column0`, `column1`; o `excel_input`
> com cabeçalho vazio nomeia `col_0`, `col_1`. **Os prefixos são diferentes** (`column` vs. `col_`).
> — comparação entre `guias-de-uso/nos/csv.md § Comportamentos importantes` e
> `guias-de-uso/nos/excel.md § Linha de cabeçalho`. Falta confirmar se é intencional.

## Entradas esperadas

Nenhuma tabular — é nó de entrada. `[INFERIDO]` Recebe a ligação de um gatilho, como o `csv_input`.

## Saídas produzidas

`[CONFIRMADO-DOC]` Dataset materializado em DuckDB, referenciado pelos nós seguintes via
`upstream_results.<nodeId>.<output_field>`. A saída inclui `row_count`.
— `guias-de-uso/nos/excel.md § Saída produzida`

```json
{"status": "completed", "row_count": 820, "output_field": "data",
 "data": {"storage_type": "duckdb"}}
```

### Conversão de tipos

`[CONFIRMADO-DOC]` — `guias-de-uso/nos/excel.md § Conversão de tipos`

| Tipo Excel | Tipo no dataset |
|---|---|
| Número inteiro | inteiro |
| Número decimal | float |
| Booleano | booleano |
| Data/hora | **string ISO 8601** |
| Texto | string |
| Vazio | `NULL` |

## Erros comuns

> ⚠️ `[CONFIRMADO-MCP]` **`.xls` (Excel 97-2003) é RECUSADO** — o nó nem tenta abrir, e a falha
> aparece na execução. Se o usuário falar em `.xls`, avise **antes**: ele precisa reabrir no Excel e
> salvar como `.xlsx`. — `describe_node('excel_input')`
>
> `[CONFIRMADO-DOC]` Mesma restrição, com o remédio explícito: *Salvar como* → *Pasta de Trabalho do
> Excel (\*.xlsx)*. — `guias-de-uso/nos/excel.md § Descrição`
>
> `[CONFIRMADO-MCP]` O MCP amplia: "Só lê **`.xlsx`/`.xlsm`**". O guia fala apenas de `.xlsx`.
> — `describe_node('excel_input')` vs. `guias-de-uso/nos/excel.md § Descrição`
>
> `[CONFIRMADO-DOC]` Note que `.xls` **está na lista de extensões aceitas no upload**
> (`.csv .tsv .xlsx .xls .json .parquet .txt`). Ou seja: **o upload aceita, o nó recusa.** O erro
> aparece só na execução. — `guias-de-uso/faq-perguntas-frequentes.md § Limites de upload`

> 🔥 `[CONFIRMADO-MCP]` **Zero à esquerda desaparece.** Coluna que a planilha guarda como NÚMERO
> chega como número: CEP `01001-000` vira `1001000`, código de barras perde o zero inicial. **O nó
> não converte para texto nem repõe o zero.** Quando o pedido envolver CEP, CNPJ, código de barras
> ou matrícula vindos de planilha, avise e trate a coluna (formatar como Texto na planilha, ou
> normalizar com `mapper` antes de usar) — senão a consulta seguinte falha e a culpa parece ser da
> API. — `describe_node('excel_input')`

`[CONFIRMADO-DOC]` **Arquivo vazio** (nenhuma linha de dados após o cabeçalho) → a execução **falha**
com erro claro. — `guias-de-uso/nos/excel.md § Comportamentos importantes`

`[CONFIRMADO-DOC]` **Download remoto** via httpx com timeout de **30 s (conexão) e 300 s (leitura)**.
O arquivo temporário é removido ao final, mesmo em caso de erro.
— `guias-de-uso/nos/excel.md § Comportamentos importantes`

`[CONFIRMADO-DOC]` **`header_row` negativo** → erro de validação.
— `guias-de-uso/nos/excel.md § Limites e guardrails`

`[CONFIRMADO-DOC]` **Múltiplas abas** → crie **um nó Excel separado para cada aba** que precisar
ler. Não há leitura multi-aba em um nó só.
— `guias-de-uso/nos/excel.md § Comportamentos importantes`

`[CONFIRMADO-DOC]` **Limites de upload:** 500 MB por arquivo, 5 GB por projeto, TTL de 30 dias,
extensões fixas e não configuráveis. Fluxo agendado que usa o arquivo **renova o "último acesso"**.
— `guias-de-uso/faq-perguntas-frequentes.md § Limites de upload`

> `[CONFIRMADO-DOC]` **Assimetria de comparação de nomes:** o Modelo de Entrada compara
> **case-insensitive**, mas o `mapper` compara de forma **exata**. Passar pela validação não garante
> que o mapeamento vai casar. — `guias-de-uso/faq-perguntas-frequentes.md § Por que meu Mapper não
> acha a coluna` + `guias-de-uso/modelos-de-entrada.md § O que é validado hoje (v1)`

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como importar a aba 'Vendas' de um Excel?"*:

```json
{"url": "shift-upload://<file_id>", "sheet_name": "Vendas", "header_row": 0, "skip_empty": true}
```

`[LACUNA]` **Não há exemplo em aula.** A transcrição `modulo-3-parte-4-saidas.txt` cobre apenas o
lado de *saída* (`xlsx_export`), não a entrada Excel. Falta descobrir se algum outro módulo
demonstra o `excel_input` na interface — em particular o rótulo literal dos campos `header_row` e
`skip_empty` na tela, que aqui estão só pelo nome técnico.
