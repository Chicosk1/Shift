# Nó `filter` — Filtrar Linhas

| | |
|---|---|
| **Tipo (MCP)** | `filter` — **alias aceito:** `filter_node` |
| **Rótulo na interface** | Filtro |
| **Categoria** | `transform` (grupo *Transformação*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | Nenhum. `consolidacao-de-nos-de-transformacao.md:19` previa `transform` step `filter`, e **`transform` não existe** — `describe_node('transform')` responde *"Tipo de nó 'transform' não encontrado no catálogo"*. Ver `divergencias.md` D3 |

## O que faz

`[CONFIRMADO-MCP]` Remove linhas do dataset que não satisfazem as condições configuradas.
Suporta operadores SQL (`eq`, `gt`, `in`, `is_null`, `contains`, etc.) com lógica AND/OR.
— `describe_node('filter')`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('filter')`
- Filtrar registros com base em um ou mais critérios.
- Remover linhas com valores nulos em campos obrigatórios.
- Selecionar um subconjunto de dados antes de carregar no destino.

`[VÍDEO]` A aula posiciona o nó como uso corriqueiro e raso: *"geralmente o Filtro você vai usar
só isso aqui"*, referindo-se a igualdade e comparação numérica. — `m3-3B:12`

`[VÍDEO]` Também é usado **depois** de um `aggregator`, para recortar o resultado agregado
(filtrar só `cidade = 'Brasília'` sobre a saída do group by). — `m3-3C:15-19`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('filter')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `conditions` | array | **sim** | Lista de condições. Duas formas: `[{field, op, value}]` **ou** `[{left: PV, operator, right: PV}]` |
| `logic` | enum (`and`, `or`) | não | Como combinar as condições. Padrão `and` |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

### ⚠️ Duas grafias para a chave do operador — fontes divergem

- `[CONFIRMADO-MCP]` A forma curta usa **`op`**: `{"field": "status", "op": "eq", "value": "ativo"}`.
  A forma longa (`left`/`right`) usa **`operator`**. — `describe_node('filter')`
- `[CONFIRMADO-DOC]` `consolidacao-de-nos-de-transformacao.md:137` escreve a forma curta com
  **`operator`**: `{ "field": "status", "operator": "eq", "value": "ativo" }`.

**Registrado nos dois lados, não resolvido.** O MCP vence por regra de precedência (usar `op` na
forma curta), mas não foi testado se o executor aceita os dois nomes. Candidata a
`divergencias.md`.

### Operadores

`[CONFIRMADO-MCP]` Confirmados nominalmente: `eq`, `gt`, `in`, `is_null`, `contains`,
`is_not_null`. A descrição encerra em *"etc."* — a lista **não é exaustiva** no contrato.

`[CONFIRMADO-DOC]` `consolidacao-de-nos-de-transformacao.md:128` afirma **16 operadores**, e usa
`gte` num exemplo. Não enumera os 16.

`[UI-OBSERVADA]` A tela mostra os operadores em português: *"é igual a"*, *"é diferente de"*,
*"contém"*, *"maior ou igual a"*. — `m3-3B:9,13,27`

`[LACUNA]` Falta a lista canônica dos 16 operadores e o mapeamento rótulo-da-UI → chave-do-MCP.
Sem isso, escrever `filter` programaticamente é tentativa e erro. Ver a lacuna de mesma natureza
em `record-id.md`.

`[LACUNA]` Falta descobrir o que **`PV`** aceita na forma `{left, operator, right}` — coluna,
literal, variável de fluxo, expressão. É a forma que permitiria comparar **duas colunas**
(ex.: `preco_novo` vs. `preco_teto`), coisa que a forma curta `{field, op, value}` não faz.

## Entradas esperadas

Um dataset tabular upstream. `[UI-OBSERVADA]` As colunas do nó anterior aparecem numa lista à
esquerda e são **arrastadas** para o campo de condição. — `m3-3B:6-7`

## Saídas produzidas

`[CONFIRMADO-MCP]` O subconjunto de linhas que satisfaz as condições, em `output_field`. O
schema não muda — filtro não cria nem remove coluna.

## Erros comuns

> ⚠️ `[VÍDEO]` **A pegadinha do AND com o mesmo campo.** A aula erra ao vivo: duas condições
> `cidade = 'Manaus'` **E** `cidade = 'Belo Horizonte'` devolvem **tabela vazia** — nenhuma linha
> tem duas cidades. O correto é `logic: or`. *"Eu não tenho ninguém que é de Manaus e Belo
> Horizonte, eu tenho quem é de Manaus e quem é de Belo Horizonte."* — `m3-3B:15-19`
>
> `[INFERIDO]` Para N valores no mesmo campo, o operador `in` resolve numa condição só, sem
> depender do `logic`. Não observado em aula.

`[VÍDEO]` Comparação numérica sobre coluna que chegou como **texto** compara lexicograficamente.
Não observado no `filter` diretamente, mas observado no `sort` (`m3-3F:16-18`) e no `math`
(`m3-3E:105-111`) com a mesma origem — base interna devolvendo número como texto. `[INFERIDO]`
Vale para o `filter`: fazer o cast num `mapper` antes.

`[LACUNA]` Comportamento do `filter` com **NULL** nas comparações não-nulas: em SQL,
`valor > 100` é desconhecido quando `valor` é NULL e a linha cai fora. Não confirmado se o nó
segue a semântica SQL pura. Importa porque decide se linhas com preço nulo somem em silêncio.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Status igual a "ativo"
{"conditions": [{"field": "status", "op": "eq", "value": "ativo"}], "logic": "and"}

// Remover nulos
{"conditions": [{"field": "valor", "op": "is_not_null"}], "logic": "and"}
```

`[VÍDEO]` Em aula, dois cenários sobre a tabela `funcionarios`:
1. `cidade = 'Manaus'` **OU** `cidade = 'Belo Horizonte'` — o caso que só funciona com `or`.
2. `cidade = 'Manaus'` **E** `salario >= 5000` — o AND legítimo, campos diferentes.
— `m3-3B:17-28`

## Observação para o piloto

É o nó do **teto de variação**: `math` cria a coluna calculada (`variacao_pct`), `filter`
descarta o que passa do teto. `[CONFIRMADO-MCP]` As duas peças existem e a ordem é essa, porque
`filter` não calcula — só compara o que já é coluna.

Três consequências diretas:

1. **A separação `math` → `filter` preserva a auditoria.** Como `math` **cria** coluna em vez de
   sobrescrever, o preço anterior e o percentual de variação continuam no dataset quando o
   `filter` recorta. Ver `math.md`.
2. **Fora-de-faixa não é o mesmo que descartar.** `filter` só remove linha. Para *rotear* as
   linhas fora da faixa a um destino de exceção — em vez de perdê-las — o nó é o `if_node`, que
   parte o dataset em dois ramos (`per_row`). Ver `lacunas.md` L5.
3. `[LACUNA]` Se o teto vier de variável de fluxo (`{{vars.teto_pct}}`), não está confirmado que
   `value` aceita interpolação, nem se a regra das aspas simples do `mapper` se aplica aqui —
   `value` é campo de dado, não expressão SQL. Ver `lacunas.md` L2 e `divergencias.md` D9.
