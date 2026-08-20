# Nó `union` — União (Union)

| | |
|---|---|
| **Tipo (MCP)** | `union` — nenhum alias declarado em `describe_node('union')` |
| **Rótulo na interface** | União `[UI-OBSERVADA]` — `m3-3I:30`; a aula também diz "o nó de União" — `m3-3I:7` |
| **Categoria** | `transform` (grupo *Transformação*) `[CONFIRMADO-MCP]` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | **`combine` com `config.mode = "union"`** `[CONFIRMADO-DOC]` (`consolidacao-de-nos-de-transformacao.md`). ⚠️ Diferente do `join`, o contrato do `union` **não** se declara legado — `describe_node('union')` não traz aviso algum. Ver divergência abaixo |

> ⚠️ **DIVERGÊNCIA fonte ↔ fonte sobre o status do nó.**
> `[CONFIRMADO-DOC]` O documento de consolidação lista `union → combine` na mesma tabela em que
> lista `join → combine`, e chama os treze nós de origem de "legados".
> `[CONFIRMADO-MCP]` Mas o `describe_node('union')` **não** contém a frase *"Versão legada — prefira
> o nó `combine`"*, que o `describe_node('join')` contém. As duas fontes registradas; nenhuma
> escolhida. Na prática: `union` funciona, e `combine`/union é o caminho recomendado para fluxo novo.

## O que faz

`[CONFIRMADO-MCP]` Combina N datasets upstream via `UNION ALL`. Alinhamento por nome (`by_name`,
com NULL para colunas ausentes) ou por posição (`by_position`). Permite adicionar coluna de origem e
deduplicação pós-união. — `describe_node('union')`

`[CONFIRMADO-DOC]` O SQL por trás de cada modo:
- `by_name` → `UNION ALL BY NAME`
- `by_position` → `UNION ALL`

E: *"Quando os datasets vêm de bancos DuckDB distintos, o nó faz `ATTACH ... READ_ONLY`
automaticamente."* — `guias-de-uso/nos/union-(juntar).md`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('union')`
- Combinar resultados de múltiplas fontes com o mesmo schema.
- Empilhar datasets de diferentes conexões numa única tabela.
- Fazer union de dois ramos paralelos **com deduplicação por chave**.

> `[VÍDEO]` Caso de negócio real dado em aula, e é o melhor argumento para o nó: no Construshow a
> tabela de cadastro de pessoas é **uma só** (`pessoa doc`); no sistema concorrente há cadastro de
> cliente, de vendedor, de funcionário e de fornecedor em **tabelas separadas**. Na migração é
> preciso empilhar as quatro numa lógica só para inserir no destino. — `m3-3I:10-12`

`[UI-OBSERVADA]` A própria tela orienta o caso oposto: *"Para combinar dados com schemas
diferentes, mas que têm uma chave em comum, use o nó de Junção Join em vez deste."* — `m3-3I:36`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('union')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mode` | enum | não | `by_name` (padrão) ou `by_position` |
| `add_source_col` | boolean | não | Se `true`, adiciona coluna com o handle de origem (ex.: `input_1`) |
| `source_col_name` | string | não | Nome da coluna de origem (padrão `_source`) |
| `dedup_keys` | array | não | Colunas-chave para deduplicação pós-união |
| `dedup_priority` | enum | não | Desempate na dedup: `first` (padrão), `last`, `input_first`, `input_last` |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

### Rótulos correspondentes na tela

`[UI-OBSERVADA]` — `m3-3I`

| Campo na tela | Parâmetro |
|---|---|
| **Conectar entradas** com botão **"+"** (`m3-3I:30-31`) | os handles `input_1..input_N` |
| **MODO DE ALINHAMENTO** — "Por nome" / "Por posição" (`m3-3I:32,44,63`) | `mode` |
| "adicionar uma coluna indicando de qual entrada veio cada linha" (`m3-3I:38`) | `add_source_col` |
| "remover duplicadas após união" (`m3-3I:39`) | `dedup_keys` |
| "se é pela primeira, pela última, da primeira entrada, da última entrada" (`m3-3I:39`) | `dedup_priority` — os quatro valores do enum, em português |

`[UI-OBSERVADA]` Texto de ajuda da própria tela, citado da aula:
- **Por nome:** *"empilha as linhas alinhando colunas pelo nome. O que faltar de um lado vira NULL.
  Recomendado em quase todos os casos."* — `m3-3I:32`
- **Por posição:** *"empilha pela ordem das colunas. Os schemas precisam ser idênticos, mesma
  quantidade de colunas, mesma ordem. Útil quando você sabe que os dois arquivos têm exatamente o
  mesmo formato."* — `m3-3I:35`

> ⚠️ **DIVERGÊNCIA de contrato em `dedup_priority` — três fontes, dois formatos.**
> - `[CONFIRMADO-MCP]` **enum**: `first` / `last` / `input_first` / `input_last`, padrão `first`.
>   — `describe_node('union')`
> - `[CONFIRMADO-DOC]` **array de objetos**: `"dedup_priority": [{"source": "input_1",
>   "priority": 1}]`. — `consolidacao-de-nos-de-transformacao.md`
> - `[CONFIRMADO-DOC]` O guia de uso do nó **não menciona dedup nenhuma** — a tabela de
>   configurações lista só `mode`, `add_source_col`, `source_col_name` e `output_field`.
>   — `guias-de-uso/nos/union-(juntar).md`
>
> Registradas as três. `[LACUNA]` Falta descobrir qual formato o executor aceita hoje. Impacto
> alto: é o campo que decide **qual** duplicata sobrevive.

## Entradas esperadas

`[CONFIRMADO-DOC]` N datasets upstream, identificados pelos handles `input_1`, `input_2`, …
`input_N`. — `guias-de-uso/nos/union-(juntar).md`

`[CONFIRMADO-DOC]` **Menos de 2 entradas → erro.** **Modo desconhecido → erro** com a lista de
modos válidos. — `guias-de-uso/nos/union-(juntar).md`

`[UI-OBSERVADA]` As entradas não vêm prontas: o instrutor clica no **"+"** dentro de "Conectar
entradas" para habilitar um terceiro ponto de conexão. — `m3-3I:30`

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma tabela única em `output_field`.
`[CONFIRMADO-DOC]` **Shape `wide`**; o resultado materializado tem `sum(N_i)` linhas.
— `guias-de-uso/nos/union-(juntar).md`

### Observabilidade — o que dá para conferir

`[CONFIRMADO-DOC]` A saída inclui `output_summary` com:
- `row_count_in` — **dict por handle**: `{"input_1": N, "input_2": M, ...}`
- `row_count_out` — soma das entradas
- `warnings` → **`schema_drift`**: emitido em modo `by_position` quando os schemas das entradas têm
  nomes ou ordem diferentes. *"Validar manualmente que o alinhamento por posição está correto."*
— `guias-de-uso/nos/union-(juntar).md`

## Erros comuns

> ⚠️ `[CONFIRMADO-DOC]` **`by_position` falha silenciosamente quando os schemas divergem** —
> *"preferir só quando a forma é controlada upstream"*. — `guias-de-uso/nos/union-(juntar).md`

`[VÍDEO]` A aula **demonstra a falha silenciosa**, e é a passagem mais valiosa do vídeo: três ramos,
cada um com uma coluna diferente removida (um sem `departamento`, um sem `email`, um sem `cidade`).
Em **Por posição** *"ele não dá erro"* — mas respeita a ordem do primeiro dataset, e o valor de
`cidade` do segundo ramo aparece **na coluna `email`**, porque estava naquela posição.
— `m3-3I:58-61`
Em **Por nome**, os mesmos dados vêm certos, com NULL onde a coluna não existe. — `m3-3I:63-66`

> `[VÍDEO]` **Atenção:** `by_position` não deu erro nem no cenário de schemas divergentes
> (`m3-3I:58`). O warning `schema_drift` é `[CONFIRMADO-DOC]` mas **não foi observado em aula** — o
> instrutor não mostra a área de warnings. `[LACUNA]` Falta descobrir se o warning aparece na tela
> ou só no `output_summary` da API. Se só na API, quem trabalha pela interface não é avisado.

`[VÍDEO]` Recomendação da aula: *"é importante sempre vir, quando vai usar o União, as colunas
sempre existir"* nos dois lados — senão a coluna presente só em um ramo vira NULL nos outros.
— `m3-3I:33-34`

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato — `describe_node('union')`:

```json
// Combinar dois datasets de clientes evitando duplicatas pelo CPF
{"mode": "by_name", "dedup_keys": ["cpf"], "dedup_priority": "first"}
```

`[CONFIRMADO-DOC]` Comportamento de `by_name` com coluna faltante —
`guias-de-uso/nos/union-(juntar).md`:

```
input_1: (ID, NOME) = (1, Alice), (2, Bob)
input_2: (ID, NOME, DEPTO) = (3, Carol, RH)
mode = by_name
→ (1, Alice, NULL), (2, Bob, NULL), (3, Carol, RH)
```

`[VÍDEO]` Da aula: `A Base Interna` → três nós **Filtro** ("Brasília", "Porto Alegre", "São
Paulo") → **União** com três entradas, simulando três tabelas de origem. Resultados 30 / 35 / 33
linhas, empilhados numa tabela só. — `m3-3I:13-30,42`

## Observação para o piloto

1. **Sempre `by_name`.** A tela recomenda (*"Recomendado em quase todos os casos"*), o doc explica
   por quê (`by_position` falha em silêncio) e a aula mostra a falha acontecendo. Numa automação de
   preço, um valor entrando na coluna errada por deslocamento posicional é o pior erro possível:
   não gera exceção, gera preço errado.
2. **Dedup pós-união é o candidato natural a idempotência** de um fluxo que reprocessa a mesma
   janela de pedidos — mas com duas ressalvas graves: o formato de `dedup_priority` está em
   divergência (acima), e a dedup só resolve duplicata **dentro da mesma execução**. Ver a análise
   completa no relatório do lote.
3. **`add_source_col` é a coluna de auditoria de graça.** Marcada, cada linha carrega de qual ramo
   veio (`input_1`, `input_2`…). Para um fluxo que decide preço, saber a procedência da linha é
   trilha de auditoria, não enfeite. A aula diz que *"não é muito utilizado"* (`m3-3I:38`) — para o
   piloto, deveria ser.
4. **`row_count_in` por handle** é o número que permite conferir "entrou o que eu esperava de cada
   fonte" antes de gravar preço. `[CONFIRMADO-DOC]`
