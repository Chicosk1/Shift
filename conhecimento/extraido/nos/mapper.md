# Nó `mapper` — Mapper (Renomear/Selecionar Colunas)

| | |
|---|---|
| **Tipo (MCP)** | `mapper` — **alias aceito:** `mapper_node` |
| **Rótulo na interface** | Mapeamento |
| **Categoria** | `transform` (grupo *Transformação*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | Nenhum. `consolidacao-de-nos-de-transformacao.md` previa fusão em `transform`, que **não existe** — ver `divergencias.md` D-MCP-3 |

## O que faz

`[CONFIRMADO-MCP]` Renomeia, seleciona ou descarta colunas do dataset upstream. Cada mapping usa
`source` (selecionar/renomear) **ou** `expression` (SQL DuckDB aplicado em runtime).
— `describe_node('mapper')`

## Quando usar

`[CONFIRMADO-MCP]`
- Renomear colunas para o padrão esperado pelo destino.
- Selecionar apenas as colunas necessárias.
- Aplicar transformações de texto via SQL (UPPER, LOWER, TRIM, REPLACE).
- Calcular campos derivados via expressão SQL arbitrária.

`[CONFIRMADO-DOC]` Caso canônico: os nomes das colunas do CSV não batem com os da tabela de
destino. — `primeiros-passos.md §3`

> `[VÍDEO]` Diferença declarada em aula frente ao nó `math`: **`math` sempre cria coluna nova;
> `mapper` sobrescreve a coluna**. — `m3-3E`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('mapper')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mappings` | array | **sim** | Lista de mapeamentos (estrutura abaixo) |
| `drop_unmapped` | boolean | não | Se `true`, remove colunas não mapeadas do resultado |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

### Estrutura de cada item de `mappings`

| Campo | Descrição |
|---|---|
| `source` | Coluna de origem. Obrigatório **se não houver `expression`** |
| `target` | Coluna de destino. **Sempre obrigatório** |
| `expression` | SQL DuckDB — **é o que o executor realmente aplica** |
| `type` | Cast opcional: `string`/`integer`/`float`/`boolean`/`date`/`datetime` |
| `expand` | Se `true`, explode a coluna STRUCT de `source` em uma coluna por campo |
| `prefix` | Prefixo opcional das colunas expandidas (ex.: `user_`) |

### Como o editor visual e a API se relacionam

`[CONFIRMADO-MCP]` **`expression` é o que o executor lê.** O editor visual compila a cadeia de
transforms (upper, lower, trim, …) em SQL e salva ali ao persistir. Para uso programático,
escrever o SQL direto.

Isso explica a relação entre o que se vê na tela `[UI-OBSERVADA]` e o que o MCP grava — são a
mesma coisa em dois níveis de abstração, não dois recursos.

### Transforms disponíveis

`[CONFIRMADO-MCP]` `upper`, `lower`, `trim`, `strip_accents`, `left`, `right`, `replace`,
`regexp_replace`, `length`, `to_date`, `to_datetime`, `to_integer`, `to_float`, `to_string`,
`to_boolean`, `coalesce`, `round`, `abs`.

`[UI-OBSERVADA]` A interface mostra rótulos em português e alguns recursos que **não aparecem
como transform no MCP**: "De Para" (com Adicionar equivalência + Fallback), "Padrão" (valor
default se nulo/vazio), "Formatar data", "Somente dígitos", "Remover caracteres especiais",
"Explodir em colunas" (= `expand`) e `Now` para DateTime. — `m3-3A`
`[LACUNA]` Falta descobrir se "De Para" e "Padrão" compilam para `CASE WHEN` e `coalesce`
respectivamente, ou se são outra coisa.

## ⚠️ Regra crítica — variáveis de fluxo em `expression`

`[CONFIRMADO-MCP]` Citado do contrato, porque erra silenciosamente:

> O token `{{vars.X}}` é substituído pelo **VALOR LITERAL** antes do SQL rodar.

- Variáveis de **texto** (`string`, `connection`, `secret`, `form_response`) **DEVEM vir entre
  aspas simples**: `'{{vars.razao_social}}'`. Sem as aspas, um valor com espaço vira
  `syntax error at or near ...`.
- Variáveis **numéricas/booleanas** entram cruas, com CAST se precisar do tipo:
  `CAST({{vars.capital}} AS DOUBLE)`.
- **Nomes de variável são case-sensitive** — usar exatamente o `name` declarado.

## Entradas esperadas

Um dataset tabular upstream. `[UI-OBSERVADA]` O nó **carrega o schema do nó anterior**
automaticamente; quando há mais de um upstream possível, define-se qual nó alimenta o
"mapear todos". — `m1:70`

## Saídas produzidas

`[CONFIRMADO-MCP]` O dataset com as colunas mapeadas em `output_field`. Com
`drop_unmapped: false` (o padrão observado nos exemplos), as colunas não mapeadas passam adiante.

`[UI-OBSERVADA]` A coluna transformada **muda de posição** — vai para o fim do dataset. Observado
em aula e potencialmente relevante para nó posterior que dependa de ordem posicional. — `m1:70`

## Erros comuns

> ⚠️ `[CONFIRMADO-DOC]` **Pegadinha nº 1 do FAQ:** o Mapper casa nome de coluna de forma
> **exata — sensível a caixa e a underscore**. `Nome` ≠ `nome` ≠ `no_me`. É a causa mais comum de
> "o Mapper não acha a coluna". Contrasta com o Modelo de Entrada, que compara case-insensitive.
> — `guias-de-uso/faq-perguntas-frequentes.md`

`[CONFIRMADO-MCP]` Variável de texto sem aspas simples em `expression` → erro de sintaxe SQL.
Ver a regra crítica acima.

`[VÍDEO]` Em aula, uma tentativa de usar `COALESCE` no modo Expressão **falhou**, e o copiloto foi
usado para corrigir. — `m3-3A`
`[LACUNA]` Falta descobrir por que falhou — `coalesce` **consta** na lista de transforms do MCP.
Possivelmente diferença entre o modo Expressão da UI e a `expression` do executor.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Maiúsculo num campo
{"mappings": [{"source": "nome", "target": "nome", "expression": "UPPER(\"nome\")"}], "drop_unmapped": false}

// Renomear
{"mappings": [{"source": "cod_cli", "target": "cliente_id"}], "drop_unmapped": false}

// Campo derivado
{"mappings": [{"target": "valor_bruto", "expression": "\"qtd\" * \"preco\""}], "drop_unmapped": false}
```

`[VÍDEO]` Em aula: nó renomeado para **"Transforma em Maiúsculos"**, aplicando transform
`Maiúsculo` na coluna `nome`. Duas estratégias mostradas — usar **"mapear todos"** (mapeamento
automático de todas as colunas), ou mapear só a coluna a transformar e marcar a opção de
**passar adiante as não incluídas**. Antes: `eduardo santos`; depois: maiúsculo. — `m1:70`

## Observação para o piloto

É o nó onde o **cálculo de margem** provavelmente vive, via `expression` com SQL DuckDB. Duas
consequências diretas:

1. Como `mapper` **sobrescreve** e `math` **cria coluna nova**, para manter o preço anterior na
   trilha de auditoria o caminho é `math` (ou `mapper` com `target` diferente), **não** sobrescrever.
2. A regra das aspas simples em `{{vars.X}}` vale para qualquer parâmetro do fluxo — incluindo um
   eventual `{{vars.dry_run}}`. Ver `lacunas.md` L2.
