# Nó `math` — Matemático (Math)

| | |
|---|---|
| **Tipo (MCP)** | `math` — nenhum alias declarado |
| **Rótulo na interface** | Matemática |
| **Categoria** | `transform` (grupo *Transformação*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | Nenhum. `consolidacao-de-nos-de-transformacao.md:18` previa `transform` step `compute`, e **`transform` não existe** no catálogo. Ver `divergencias.md` D3 |

## O que faz

`[CONFIRMADO-MCP]` Adiciona colunas calculadas ao dataset via expressões SQL avaliadas pelo
DuckDB. Suporta qualquer função aritmética, de data ou string disponível no DuckDB.
— `describe_node('math')`

## ⚠️ A regra que define o nó — e o separa do `mapper`

> `[CONFIRMADO-MCP]` *"Cada item de `expressions` **adiciona uma nova coluna** ao dataset com o
> resultado da expressão SQL."* — `describe_node('math')`, campo *Transforms disponíveis*
>
> `[VÍDEO]` A aula diz o mesmo com as duas metades juntas: *"a matemática ela sempre vai gerar
> uma nova coluna. Enquanto que o, o mapeamento ele vai se tornar aquela coluna"* … *"no momento
> que eu tô criando cliente_padronizado vai passar pra frente cliente **e** cliente_padronizado"*.
> — `m3-3E:146-147`, repetido em `m3-3E:167`

**Consequência para o piloto:** preservar o preço anterior para auditoria depende de `math`, não
de `mapper`. Um `mapper` que grava em `target: "preco"` **perde** o valor antigo no mesmo passo.

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('math')`
- Calcular um campo derivado como `total = quantidade * preco`.
- Converter ou formatar datas com funções SQL.
- Concatenar ou transformar strings via expressão SQL.

`[VÍDEO]` O argumento de projeto dado em aula, que vale registrar porque é diretriz e não
recurso: *"Isso evita que você fique fazendo expressão matemática na consulta, traz os dados que
você tem, se você precisa gerar algum valor, gera por aqui que é muito mais rápido e muito mais
seguro e você consegue auditar e colocar mais regras aqui"*. — `m3-3E:59`

`[VÍDEO]` Para manipulação de **texto**, a aula prefere o `mapper`: *"A parte de manipulação de
texto eu prefiro lá, mas se você já tiver fazendo um tratamento aqui e você vai precisar fazer
isso, você já pode fazer aqui"*. — `m3-3E:143`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('math')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `expressions` | array | **sim** | `[{target_column, expression}]`, onde `expression` é SQL DuckDB |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

`[CONFIRMADO-MCP]` *"Expressões podem referenciar qualquer coluna existente pelo nome."*
— `describe_node('math')`

**O contrato tem só esses dois campos.** Tudo o que a interface oferece a mais compila para
`expression`, exatamente como no `mapper`.

### As quatro abas da interface

`[UI-OBSERVADA]` Cada expressão adicionada (**"+ Adicionar expressão"**) tem um nome de coluna e
uma aba:

| Aba | O que oferece | Compila para |
|---|---|---|
| Cálculo (padrão) | Termos arrastados da lista de colunas ou **valor fixo**, com operador entre eles (*"tem várias opções matemáticas"*) | aritmética SQL |
| **Condição** | `SE / ENTÃO`, **"+ Adicionar senão se"**, e um `SENÃO` final. O resultado pode ser valor, texto ou campo lincado | `CASE WHEN … ELSE … END` `[INFERIDO]` |
| **Texto** | Maiúscula, minúscula, primeiros N caracteres, **"juntar com outra coluna"** (concatenação), via **"+ Adicionar transformação"** | `UPPER`, `LOWER`, `LEFT`, `CONCAT` `[INFERIDO]` |
| **Avançado** | Campo livre de expressão com um dropdown **"Funções"** (traz *"diferença em dias"*, *"data de hoje"*) | a `expression` crua |

— `m3-3E:22-23,29-33,120-133,140-152,153-160`

### O "envolvimento de cálculo" — e a prova de que a UI compila para SQL

`[UI-OBSERVADA]` Na aba Cálculo há um dropdown cujo padrão é **"Sem ajuste"**, com opções
*"Arredondar (2 casas)"*, inteiro para cima, para baixo, absoluto, raiz quadrada. — `m3-3E:37-39`

`[UI-OBSERVADA]` **A tela mostra o SQL gerado:** *"ele até mostra a expressão que gera … o que
que ele vai gerar por baixo dos panos é basicamente … round preço unitário vezes quantidade 2"* —
isto é, `round(preco_unitario * quantidade, 2)`. — `m3-3E:41`

Isso fecha a mesma questão que `mapper.md` levanta: interface e contrato são **dois níveis de
abstração do mesmo campo**, não dois recursos. Aqui a aula exibe o SQL na tela.

## Entradas esperadas

Um dataset tabular upstream, com as colunas **já no tipo certo**.

> ⚠️ `[VÍDEO]` **Coluna de texto quebra o nó.** Com `preco_unitario`, `quantidade` e `desconto`
> chegando como texto, a execução falhou com *"mensagem de erro vermelha … indicando
> incompatibilidade de tipos nas funções"*. O conserto foi voltar ao `mapper` e declarar
> `desconto` Decimal, `quantidade` Inteiro, `preco_unitario` Decimal. — `m3-3E:105-111`
>
> `[UI-OBSERVADA]` O indicador de tipo na grade de saída muda de **`T`** (texto) para **`#`**
> (número) quando o cast pega — jeito rápido de conferir na tela. — `m3-3E:111`

## Saídas produzidas

`[CONFIRMADO-MCP]` O dataset original **inteiro**, mais uma coluna nova por item de
`expressions`, em `output_field`.

## Erros comuns

### 1. Não se pode usar no mesmo nó uma coluna criada nele

> `[VÍDEO]` *"eu não consigo usar uma coluna sem ter gerado ela antes"* — a aula precisou de um
> **segundo nó `math`** encadeado para classificar a faixa a partir de `total_liquido`, criado no
> primeiro. — `m3-3E:66-69`

`[INFERIDO]` As expressões de um mesmo `math` são avaliadas na mesma projeção SQL, então nenhuma
vê o resultado da outra. **Consequência de projeto:** cadeias de cálculo dependente = um `math`
por degrau.

### 2. NULL propaga em silêncio

> `[VÍDEO]` *"os que tão sem desconto … total_liquido, os que tão sem desconto ele tá, ele tá
> deixando null"*. Linhas com `desconto` nulo produziram `total_liquido` **nulo**, sem erro.
> — `m3-3E:83`

O conserto mostrado é a montante, no `mapper`: `[UI-OBSERVADA]` transform **"Padrão"** com valor
`0` em `preco_unitario` e `desconto`, e `1` em `quantidade` (*"a quantidade, se eu tiver uma
venda, não pode ser zero"*). — `m3-3E:89-93`

**Para o piloto isto é grave:** um custo nulo produz margem nula, a linha passa pelo `filter` de
teto sem disparar nada e o preço é escrito com base em cálculo vazio. Tratar nulo **antes** do
`math` não é higiene, é controle.

### 3. Erro de tipo por cast faltando

Ver *Entradas esperadas*. — `m3-3E:105-111`

### 4. Refazer ligação apaga a configuração

`[VÍDEO]` Ao desconectar e reconectar o upstream, as expressões do nó **foram perdidas** e
tiveram de ser redigitadas. — `m3-3E:99-103`. `[LACUNA]` Falta descobrir se é comportamento
esperado (schema de entrada mudou) ou defeito da interface. Não reproduzido.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
{"expressions": [{"target_column": "valor_total", "expression": "quantidade * preco"}]}
```

`[CONFIRMADO-DOC]` **Margem, do próprio documento oficial** — é o exemplo canônico do step
`compute`, e a única ocorrência do termo "margem" em todo o acervo bruto:

```json
{
  "type": "math",
  "expressions": [
    { "target_column": "margem",       "expression": "(preco - custo) / preco" },
    { "target_column": "desconto_pct", "expression": "desconto / preco * 100" }
  ],
  "output_field": "data"
}
```
— `consolidacao-de-nos-de-transformacao.md:98-105`

`[VÍDEO]` Em aula, sobre uma tabela de vendas (`cliente`, `produto`, `desconto`,
`preco_unitario`, `quantidade`):

```json
// nó Matemática 1 — arredondamento de 2 casas vindo do dropdown "Arredondar (2 casas)"
{"expressions": [
  {"target_column": "total_bruto",   "expression": "round(preco_unitario * quantidade, 2)"},
  {"target_column": "total_liquido", "expression": "round(preco_unitario * quantidade - desconto, 2)"}
]}
```
Conferência narrada: `3500 * 2 = 7000`; `49,90 * 10 = 499`; `7000 - 200 = 6800`. — `m3-3E:23-58`

```
// nó Matemática 2 — aba Condição, coluna "faixa"
SE total_liquido > 1000        ENTÃO 'Alto'
SENÃO SE total_liquido > 500   ENTÃO 'Médio'
SENÃO                                'Baixo'
```
— `m3-3E:117-138`. `[UI-OBSERVADA]` A aula compara o recurso ao "De Para" do `mapper`: *"é bem
parecido com lá na transformação … como o DE PARA lá, né, mas aqui você consegue encadear de
maneira mais visual"*. — `m3-3E:139`

`[VÍDEO]` Na aba Avançado, uma expressão colada à mão, descrita pelo narrador como
`trim` + `upper` no cliente, `concat` com `', R$ '` e `round(quantidade * preco_unitario -
desconto, 2)`, produzindo `ANA COSTA - 6800…`. — `m3-3E:160-165`. `[LACUNA]` O SQL exato não
aparece na transcrição, só a narração do que ele faz.

## Observação para o piloto

Este é **o nó do cálculo de margem**, e por três razões que se reforçam:

1. `[CONFIRMADO-MCP]` **Cria coluna, não sobrescreve.** `preco_atual`, `preco_sugerido`,
   `margem_atual`, `variacao_pct` coexistem na mesma linha — a trilha de auditoria sai de graça
   do próprio dataset. `mapper` no mesmo lugar destruiria o valor de origem.
2. `[CONFIRMADO-DOC]` A fórmula `(preco - custo) / preco` está escrita na documentação oficial
   como exemplo de `math`. Isso confirma **o nó e a forma**, não os nomes de campo do ERP — onde
   a margem vive continua sendo `lacunas.md` L4.
3. `[VÍDEO]` **Um `math` por degrau.** `margem_atual` → `preco_sugerido` → `variacao_pct` são
   três dependências em cadeia, logo **três nós `math`** encadeados (`m3-3E:66-69`), não três
   expressões num nó. Isso muda o desenho do canvas.

Ordem obrigatória a montante: `mapper` (cast + "Padrão" para nulos) → `math`. Sem isso, o nó
quebra por tipo (`m3-3E:105`) ou devolve nulo em silêncio (`m3-3E:83`).

`[LACUNA]` Não confirmado se `{{vars.X}}` funciona em `expression` do `math` como funciona no
`mapper`. É a **mesma** `expression` SQL DuckDB, então `[INFERIDO]` a regra das aspas simples
vale igual — mas isso não foi verificado no contrato do `math`, que não menciona variáveis. Ver
`divergencias.md` D9.
