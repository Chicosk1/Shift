# Nó `sort` — Ordenação (Sort)

| | |
|---|---|
| **Tipo (MCP)** | `sort` — nenhum alias declarado |
| **Rótulo na interface** | Ordenar |
| **Categoria** | `transform` (grupo *Transformação*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | Nenhum. `consolidacao-de-nos-de-transformacao.md:20` previa `transform` step `sort`, e **`transform` não existe** no catálogo. Ver `divergencias.md` D3 |

## O que faz

`[CONFIRMADO-MCP]` Ordena o dataset upstream por uma ou mais colunas com direção (ASC/DESC) e
posição de nulos configurável. Suporta limite dos N primeiros **após** a ordenação.
— `describe_node('sort')`

`[CONFIRMADO-DOC]` *"Um limite opcional restringe a saída aos N primeiros registros após a
ordenação — útil para construir listas 'top-N' sem rodar uma query separada."*
— `guias-de-uso/nos/sort-(ordenar).md:10`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('sort')`
- Ordenar registros por data, valor ou qualquer coluna.
- Obter os N maiores ou menores valores de um dataset.
- **Garantir ordem consistente antes de um nó de carga.**

> `[VÍDEO]` **Diretriz de arquitetura dada em aula:** ordenar no Shift, não no banco.
> *"É preferível que se você precise ordenar, você ordene por esse nó. Se você ordena pela
> consulta, se torna muito mais lento o processo… Trazer a consulta com o mínimo de parâmetros
> que você puder… se tiver 1 milhão de linhas você trazer com o where que você precisa, mas não
> usar o order by, né. Porque lá no banco vai pesar esse processo."* — `m3-3F:50-56`
>
> ⚠️ **Isto conflita com o teto de 50M linhas do documento oficial** (abaixo). A aula recomenda
> mover a ordenação para dentro do Shift; o guia avisa que o Shift satura memória em volume
> grande e sugere *"particione antes de ordenar"*. As duas afirmações se contradizem no regime de
> volume alto. **Registrado nos dois lados, não resolvido** — candidata a `divergencias.md`.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('sort')`; detalhe de padrão por
`guias-de-uso/nos/sort-(ordenar).md:36-43`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `sort_columns` | array | **sim** | — | `[{column, direction, nulls_position?}]` |
| `sort_columns[].column` | string | **sim** | — | Nome da coluna |
| `sort_columns[].direction` | string | não | `asc` | `asc` ou `desc` |
| `sort_columns[].nulls_position` | string | não | `last` em `asc` / `first` em `desc` | `first` ou `last` |
| `limit` | int | não | — | Mantém só os N primeiros **após** a ordenação |
| `output_field` | string | não | `data` | Campo de saída |

`[UI-OBSERVADA]` A tela usa `ASC`/`DESC` literalmente e o campo do limite se chama **"LIMITE DE
LINHAS (OPCIONAL)"**, com a descrição *"quando informado, retorna apenas n primeiros registros
após a ordenação"*. — `m3-3F:14,42,44`

`[VÍDEO]` Sobre o limite: *"geralmente você não vai precisar usar"* / *"é um recurso que tem, não
é muito utilizado"*. — `m3-3F:41,49`

`[LACUNA]` `nulls_position` **não aparece na interface** em nenhum momento de `m3-3F`. Falta
descobrir se a UI expõe o controle ou se ele é só de API — importa porque a política de nulos
decide se linhas sem preço vão para o topo ou o fim do lote.

## Entradas esperadas

Um dataset tabular upstream.

> ⚠️ `[VÍDEO]` **Ordenação de número em coluna de texto ordena como texto.** A aula ordena `id`
> ASC e o resultado sai fora de ordem: *"É que o ID tá como texto, né. Então por isso que ele
> ficou assim"*. O conserto foi um `mapper` entre a origem e o `sort`, forçando `id` → Inteiro:
> *"E agora vai funcionar melhor o nosso ordenar aqui."* — `m3-3F:16-26`

## Saídas produzidas

`[CONFIRMADO-MCP]` O mesmo dataset reordenado, em `output_field`. O schema não muda.

`[CONFIRMADO-DOC]` A saída inclui `output_summary` com `row_count_in` / `row_count_out` e
`warnings` — **lista sempre vazia neste nó**: *"sort não tem heurísticas com warning
automático"*. — `sort-(ordenar).md:61-67`

Isto contrasta com o `record_id`, que emite `non_deterministic_without_order_by`. Ver
`record-id.md`.

## Notas de performance

`[CONFIRMADO-DOC]` — `sort-(ordenar).md:45-51`
- **Shape `wide`.** `ORDER BY` em DuckDB usa ordenação **em memória**, com fallback para
  spillover em disco quando o dataset não cabe.
- **Datasets acima de 50M linhas tendem a saturar a memória**; nesse caso preferir `LIMIT` ou
  **particionar antes de ordenar**.
- Sem `limit`, o nó **materializa o dataset inteiro** reordenado — custo proporcional a
  `N * log(N)`.

## Erros comuns

`[CONFIRMADO-DOC]` Guardrails que viram erro — `sort-(ordenar).md:53-59`:
- `sort_columns` vazio → `NodeProcessingError`.
- `column` vazio em qualquer item → erro.
- `limit` não inteiro ou negativo → erro.

`[VÍDEO]` Ordenar coluna numérica que chegou como texto — ver *Entradas esperadas*.
— `m3-3F:16-18`

`[VÍDEO]` Ordenação por múltiplas colunas confunde quem espera dois critérios independentes:
`nome ASC` + `cidade DESC` ordena por nome e usa cidade só para desempate. A aula prevê a
confusão (*"vai dar uma confusão né, de ordenação"*) e confere o resultado: nome começando em A,
e a última cidade sendo Belo Horizonte. — `m3-3F:34-39`

`[LACUNA]` Não confirmado se o `sort` **sobrevive** ao nó seguinte, isto é, se a ordem é
garantida ao atravessar um nó que não declara ordenação. Em SQL, ordem não é propriedade
preservada entre operações. O contrato diz *"garantir ordem consistente antes de um nó de carga"*
`[CONFIRMADO-MCP]`, o que sugere que sim para o nó imediatamente seguinte, mas não é afirmação
explícita.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
{"sort_columns": [{"column": "data_pedido", "direction": "desc"}], "limit": 100}
```

`[CONFIRMADO-DOC]` Exemplo com resultado tabelado em `sort-(ordenar).md:12-32`:
`sort_columns=[{GRUPO, asc}, {VALOR, desc}]` sobre 4 linhas → grupo A antes de B, e dentro de
cada grupo o valor do maior para o menor.

`[VÍDEO]` Em aula, sobre `funcionarios`: `id ASC`, depois `id DESC`, depois
`nome ASC` + `cidade DESC`, e por fim `limit = 5` — que devolveu os 5 primeiros **da ordenação**,
não os 5 primeiros da tabela: *"Então aqui é importante que é após a ordenação."* — `m3-3F:14-48`

## Observação para o piloto

O `sort` **não** entra no cálculo de margem. Onde ele importa é em três pontos de borda:

1. `[CONFIRMADO-MCP]` **Ordem antes da carga.** *"Garantir ordem consistente antes de um nó de
   carga"* é caso de uso declarado. Para o `bulk_insert` do ajuste de preço, ordenar por chave
   torna a escrita — e o diff da auditoria — reproduzível.
2. `[CONFIRMADO-MCP]` **`limit` como contenção de volume**, distinta do disjuntor: `sort` +
   `limit` corta o lote a N linhas **em silêncio**, sem avisar que cortou. Um disjuntor precisa
   *detectar* o estouro, não escondê-lo — para isso é `aggregator` + `if_node`
   (`decision_mode: single`). Ver `aggregator.md` e `lacunas.md` L5.
3. `[CONFIRMADO-DOC]` O teto de 50M linhas não ameaça o piloto (pedidos de 5 em 5 minutos), mas
   ameaça qualquer carga histórica inicial. E aí a orientação do documento (*particione antes*)
   vale mais que a da aula (*ordene sempre no Shift*).
