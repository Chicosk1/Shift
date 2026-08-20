# Nó `csv_input` — Entrada CSV

| | |
|---|---|
| **Tipo (MCP)** | `csv_input` |
| **Rótulo na interface** | CSV |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Lê um arquivo CSV local ou remoto (HTTP/S3) e **materializa o conteúdo em uma
tabela DuckDB**. Suporta upload via interface. — `describe_node('csv_input')`

## Quando usar

`[CONFIRMADO-MCP]`
- Importar dados de um arquivo CSV para o fluxo.
- Ler arquivo CSV enviado pelo usuário via upload.
- Consumir CSV de URL remota (HTTP, S3, GCS).

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('csv_input')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `url` | string | **sim** | — | Caminho local ou URL. Aceita `shift-upload://<file_id>` para uploads |
| `delimiter` | string | não | `,` | Separador de colunas |
| `has_header` | boolean | não | `true` | Se a primeira linha é o cabeçalho |
| `encoding` | string | não | `utf-8` | Encoding do arquivo |
| `null_padding` | boolean | não | `true` | Preenche colunas faltantes com NULL |
| `max_rows` | number | não | — | Limite de linhas a ler |
| `output_field` | string | não | `data` | Nome do campo de saída |

### As 5 origens de arquivo na interface

`[UI-OBSERVADA]` A interface oferece cinco formas de apontar o arquivo, todas resolvendo para
`url`: **link** (URL), **arquivos do projeto**, **arquivos da área**, **enviar agora** (upload
manual) e **variável** — esta última para o caso de o arquivo chegar por API. — `m1:66`

`[CONFIRMADO-DOC]` A escolha entre elas tem tabela de recomendação em
`guias-de-uso/variaveis-e-arquivos-no-runtime.md`: URL/Path para S3/FTP e automação sem humano;
"Do projeto", "Enviar" e "Variável" para os demais casos.

> ⚠️ `[CONFIRMADO-DOC]` **Variável tipo Arquivo não funciona em fluxo agendado (`cron`)** — não
> há ninguém para fazer upload. Use URL/Path, ou dispare por API com `variable_values`.
> — `guias-de-uso/variaveis-e-arquivos-no-runtime.md § Workflows agendados (cron)`

### Restrições declaradas

`[CONFIRMADO-DOC]` `delimiter` tem de ter **exatamente 1 caractere**, e a lista de encodings é
fechada. — `guias-de-uso/nos/csv.md`

## Entradas esperadas

Nenhuma tabular — é nó de entrada. `[VÍDEO]` Recebe a ligação de um gatilho. — `m1:65-66`

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma tabela DuckDB no campo `output_field` (padrão `data`), com uma coluna por
coluna do CSV.

## Erros comuns

`[CONFIRMADO-DOC]` Leitura em **streaming** via DuckDB — arquivo grande não é carregado inteiro
em memória. Há retentativa configurável. — `guias-de-uso/nos/csv.md`

`[CONFIRMADO-DOC]` **Limites de upload:** 500 MB por arquivo, 5 GB por projeto, TTL de 30 dias,
extensões fixas (`.csv .tsv .xlsx .xls .json .parquet .txt`) e **não configuráveis**. Pegadinha
declarada: fluxo agendado que usa o arquivo **renova o "último acesso"**.
— `guias-de-uso/faq-perguntas-frequentes.md`

`[CONFIRMADO-DOC]` Se o fornecedor do arquivo costuma errar nome de coluna, vincular um **Modelo
de Entrada**: o Shift valida **antes** de processar e falha com mensagem clara em vez de erro
críptico adiante. — `primeiros-passos.md §2`

> `[CONFIRMADO-DOC]` **Assimetria de comparação de nomes:** o Modelo de Entrada compara
> **case-insensitive**, mas o `mapper` compara de forma **exata**, sensível a caixa e underscore.
> Passar pela validação não garante que o mapeamento vai casar.
> — `guias-de-uso/faq-perguntas-frequentes.md`

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como importar um CSV com separador ponto e vírgula?"*:

```json
{"url": "shift-upload://<file_id>", "delimiter": ";", "has_header": true, "encoding": "utf-8"}
```

`[VÍDEO]` Em aula: arquivo `funcionarios.csv` com **300 linhas**, enviado por upload manual.
Delimitador e encoding foram deixados no padrão porque o arquivo já era vírgula e UTF-8. O
resultado foi conferido com Executar e visualização em tabela antes de seguir. — `m1:66-69`
