# Nó `aggregator` — Aggregator (Group By)

| | |
|---|---|
| **Tipo (MCP)** | `aggregator` — nenhum alias declarado |
| **Rótulo na interface** | Agregador |
| **Categoria** | `transform` (grupo *Transformação*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | Nenhum. `consolidacao-de-nos-de-transformacao.md:26` previa o nó `aggregate` com `config.mode = "group_by"`, e **`aggregate` não existe** — `describe_node('aggregate')` responde *"Tipo de nó 'aggregate' não encontrado"*. `aggregator` **é** o nó vigente, não o legado. Ver `divergencias.md` D3 |

## O que faz

`[CONFIRMADO-MCP]` Agrupa linhas e calcula métricas (SUM, AVG, COUNT, MAX, MIN) por grupos de
colunas, **ou** junta os valores de uma coluna numa única string separada por delimitador
(`string_agg`). Com `group_by` vazio, agrega todas as linhas **num único registro**.
— `describe_node('aggregator')`

`[VÍDEO]` A definição dada em aula: *"É basicamente um group by"* … *"o agregador ele permite
você fazer agregações como se fosse um GROUP BY do banco de dados"*. — `m3-3C:3,37`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('aggregator')`
- Calcular totais ou médias agrupadas por uma ou mais colunas.
- Contar registros por categoria.
- Sumarizar dados antes de carregar no destino.
- Juntar os valores de uma coluna numa única linha separada por vírgula (ex.: lista de códigos
  para um `IN` de SQL ou para um texto).

`[VÍDEO]` Uso que não está no contrato e importa: **buscar o próximo ID de uma tabela**.
`group_by` vazio + `MAX(id)` devolve uma linha com o último ID, que depois alimenta o
`record_id`. — `m3-3H:10,26-28`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('aggregator')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `group_by` | array | não | Colunas para agrupar. **Vazio = agrega tudo num registro** |
| `aggregations` | array | **sim** | `[{operation, column, alias}]` |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

### Estrutura de cada item de `aggregations`

| Campo | Descrição |
|---|---|
| `operation` | `sum` \| `avg` \| `count` \| `max` \| `min` \| `string_agg` |
| `column` | Coluna alvo. **Obrigatória em `string_agg`** |
| `alias` | Nome da coluna de resultado |
| `separator` | Só em `string_agg`: texto entre os valores. Padrão `", "` |
| `distinct` | Só em `string_agg`: booleano, remove repetidos — **não preserva a ordem original** |

`[UI-OBSERVADA]` Rótulos na tela: **"Adicionar coluna de agrupamento"**, **"Adicionar
agregação"**, e no cabeçalho do campo de grupo **"AGRUPAR POR"**. O alias é digitado à mão — a
aula usa `count_nome`, `sum_salario`, `avg_salario`. — `m3-3C:6,8,29`; `m3-3H:27`

### ⚠️ `count` precisa de coluna? Fontes divergem

- `[CONFIRMADO-MCP]` O exemplo do contrato conta **sem `column`**:
  `{"operation": "count", "alias": "total"}`.
- `[CONFIRMADO-DOC]` `consolidacao-de-nos-de-transformacao.md:410` conta com
  `"column": "*"`.
- `[VÍDEO]` Em aula, conta **uma coluna nomeada** (`nome`). — `m3-3C:7-8`

**Registrado nos três lados.** As três formas provavelmente coexistem, mas atenção: contar uma
coluna nomeada é `COUNT(col)`, que em SQL **ignora NULL**, enquanto `COUNT(*)` conta a linha.
`[INFERIDO]` — não testado no nó. Para contagem de disjuntor, isso é a diferença entre o número
certo e um número menor em silêncio.

## Entradas esperadas

Um dataset tabular upstream.

`[VÍDEO]` Se a coluna a agregar chegou como **texto**, `MAX`/`SUM` dão resultado errado ou
quebram — a aula insere um `mapper` só para forçar `id` → Inteiro e `salario` → Decimal antes do
agregador. — `m3-3H:15-21`

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma linha por combinação de `group_by`, com as colunas de `group_by` mais uma
coluna por `alias`. Com `group_by: []`, **uma linha só**.

`[VÍDEO]` As colunas que não estão em `group_by` nem em `aggregations` **não passam adiante** — a
saída do agregador é só grupo + métricas. Observado implicitamente ao longo de `m3-3C:10-19`,
onde o `filter` seguinte só tem `cidade`, `departamento` e `count_nome` para filtrar.
`[INFERIDO]` É a semântica normal de GROUP BY.

`[VÍDEO]` Grupos com valor **nulo** aparecem como um grupo próprio: *"ainda nós temos aquele
nulo, né? Eh, mas tudo bem"*. — `m3-3C:14`

## Erros comuns

`[VÍDEO]` Agregar coluna de texto sem cast prévio — resolver com `mapper` a montante.
— `m3-3H:15-21`

`[VÍDEO]` Confundir `group_by` vazio com `group_by` preenchido no caso do MAX: *"se eu coloco
alguma coluna aqui ele vai fazer o MAX e agrupar pela coluna"*. Para "o maior ID da tabela
inteira" o `group_by` tem de ficar **vazio**. — `m3-3H:26-28`

`[LACUNA]` Não há `count_distinct` na lista de operações. `distinct` existe **só** dentro de
`string_agg`. Falta descobrir como contar valores distintos — provavelmente `aggregator` sobre a
saída de um `deduplication`, mas isso é `[INFERIDO]`.

`[LACUNA]` Não há mediana, percentil nem desvio padrão. `[INFERIDO]` A família `stat_*` cobre
parte disso (ver `lacunas.md` L19), mas não foi verificado.

## Limitação declarada em aula

> `[VÍDEO]` **Não há arredondamento no SUM.** A aula identifica a falta ao vivo, anota como
> pedido de melhoria (*"Nó Agregador: no SUM poder arredondar"*) e o contorno é arredondar
> depois: *"depois obviamente você poderia arredondar isso ali para a frente"*. — `m3-3C:30-32`
>
> `[INFERIDO]` O contorno é um `math` a jusante com `ROUND(sum_salario, 2)`. Note que isso
> **cria outra coluna** em vez de corrigir a existente — ver `math.md`.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Contar registros por status
{"group_by": ["status"], "aggregations": [{"operation": "count", "alias": "total"}]}

// Todos os códigos do IBGE numa linha, separados por vírgula
{"group_by": [], "aggregations": [{"operation": "string_agg", "column": "IBGE", "separator": ", ", "alias": "ibge_lista"}]}
```

`[VÍDEO]` Em aula, três configurações sobre `funcionarios`:

| `group_by` | `aggregations` | Resultado narrado |
|---|---|---|
| `["cidade"]` | `count` de `nome` → `count_nome` | 27 em Recife, 32 em Fortaleza, 33 em Brasília |
| `["cidade","departamento"]` | idem | Brasília/Marketing = 4, Brasília/RH = 6 |
| `["departamento"]` | `sum` de `salario` → `sum_salario`, depois `avg` → `avg_salario` | média ~8 mil |

— `m3-3C:9-11,19,27-36`

`[VÍDEO]` E o padrão do próximo ID, que é o que interessa ao piloto:

```json
// "qual o último id da tabela" — group_by VAZIO
{"group_by": [], "aggregations": [{"operation": "max", "column": "id", "alias": "max_id"}]}
```
Resultado em aula: `max_id = 301`. — `m3-3H:26-28`

## Observação para o piloto

É a **metade do disjuntor**. `[CONFIRMADO-MCP]` `group_by: []` + `count` reduz o lote a uma
linha com o número de itens que seriam alterados. A outra metade é o `if_node` com
`decision_mode: 'single'`, que *"avalia SÓ a 1ª linha e envia a tabela inteira por um único
ramo"* — decisão de lote, não de linha. Ver `lacunas.md` L5.

Três ressalvas:

1. `[CONFIRMADO-MCP]` **O agregador descarta as colunas de dado.** Contar num agregador e depois
   escrever as linhas exige recolar a contagem no dataset original. `[VÍDEO]` O caminho mostrado
   em aula para isso é `combine` em modo **cross join** — a agregação de uma linha é
   multiplicada em todas as linhas do dataset. — `m3-3H:43-50`
2. `[VÍDEO]` Nessa colagem, *"o nó Combinar, quando você passa uma tabela que tem mais de um dado
   ele vai pegar sempre o primeiro"*. — `m3-3H:59`. `[LACUNA]` Frase ambígua e não verificada
   contra o contrato do `combine`; é do lote de `combine`, não deste.
3. `[LACUNA]` **Não existe nó de falhar/abortar** (L10). O agregador conta, o `if_node` decide,
   mas o ramo "estourou o limite" não tem para onde ir senão terminar sem escrever. Ver o item
   sobre L10 em `lacunas.md`.
