# Nó `sql_database` — Extração SQL / "SELECT"

| | |
|---|---|
| **Tipo (MCP)** | `sql_database` |
| **Aliases (MCP)** | `extractNode` |
| **Rótulo no contrato do MCP** | Extração SQL |
| **Rótulo na interface (aula)** | **SELECT** — *"nós temos o nó SELECT"* `m3p2b:97` `[UI-OBSERVADA]` |
| **Categoria** | `input` `[CONFIRMADO-MCP]` — mas na **UI aparece dentro do grupo *Banco de Dados*** `m3p2b:97-98` `[UI-OBSERVADA]` |
| **Risco** | **`read_only`** `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

> **É o nó mais importante do piloto de margem.** Não porque escreve — ele não escreve — mas
> porque é o **único lugar do catálogo com marca d'água nativa**. Toda a estratégia de "ler só
> pedidos novos" mora nos três campos da seção *Leitura incremental*, abaixo.

## O que faz

`[CONFIRMADO-MCP]` Extrai dados de um banco de dados externo via query `SELECT`. Suporta
**streaming** e **leitura particionada paralela** para tabelas grandes.
— `describe_node('sql_database')`

`[VÍDEO]` Em aula, descrito como o nó "óbvio" de banco: *"eu posso simplesmente, por variável ou
alguma coisa, colocar aqui um SELECT que vai me trazer dados do banco obviamente que eu
selecionei"*. — `m3p2b:101`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('sql_database')`
- Extrair dados de banco externo (PostgreSQL, SQL Server, MySQL, **Oracle**) via query.
- Ler uma tabela inteira (`table_name`) ou filtrar com `WHERE` (`query`).
- Usar **particionamento paralelo** para acelerar extração de tabelas grandes.

`[VÍDEO]` Orientação de aula, que é uma regra de escolha entre este nó e o `sql_script`:

> *"a orientação que eu dou para sempre que for um SQL pesado ou qualquer nó que você precise
> executar um SQL mas que não dependa de parâmetros, usar o nó SELECT."* — `m3p2b:105`

> *"a única diferença do SELECT pro SQL Script (…) é porque o SELECT ele foi feito com uma
> estratégia de SELECT em bancos diferente, para ele ter uma alta performance. Então por isso que
> para buscar dados mais pesados, que tragam mais dados, eu sempre oriento a usar o nó SELECT."*
> — `m3p2b:107`

`[INFERIDO]` A "estratégia diferente" narrada em aula é provavelmente o **streaming por chunk**
(`chunk_size`, padrão 50.000) mais o **particionamento paralelo** (`partition_on` /
`partition_num`) que o contrato expõe e que o `sql_script` não tem. A aula não nomeia o mecanismo.

## ⚠️ Armadilhas declaradas no contrato

`[CONFIRMADO-MCP]` Citadas literalmente — `describe_node('sql_database')`:

> - Placeholder dentro de **COMENTÁRIO** conta como parâmetro obrigatório: um `:cnpj` escrito num
>   `-- filtra por :cnpj` vira bind exigido. Ou remova o `:` do comentário, ou forneça o parâmetro.
> - Parâmetro que resolve para `None` é **RECUSADO antes de tocar o banco**. Todo `:nome` da query
>   precisa de um valor de verdade em `parameters`.
> - Os nomes **`incremental_from`, `shift_p_lo` e `shift_p_hi` são reservados pelo motor** (leitura
>   incremental / particionamento) e são **recusados** como nome de parâmetro.

> ⚠️ **A terceira é a que morde o piloto.** Se alguém escrever a query incremental à mão como
> `WHERE data_emissao > :incremental_from` esperando alimentar o bind manualmente, o nó **recusa o
> nome**. O `incremental_from` é injetado **pelo motor**, não pelo usuário.
> `[LACUNA]` **Falta descobrir se a query precisa referenciar `:incremental_from` explicitamente**
> ou se o motor reescreve o `WHERE` sozinho a partir de `incremental_column`. O contrato declara os
> nomes como reservados mas não mostra um exemplo de query incremental. Ver L31.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('sql_database')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `connection_id` | string | **sim** | — | UUID do conector SQL de origem |
| `query` | string | não¹ | — | Query `SELECT` a executar. Aceita placeholders `:nome` resolvidos via `parameters` — **bind de verdade, sem interpolação de texto** |
| `table_name` | string | não¹ | — | Nome da tabela a ler integralmente (alternativa a `query`) |
| `parameters` | object | não | — | Mapa `{nome: ParameterValue}` para os `:nome` da query — valor fixo, variável do fluxo ou campo do passo anterior |
| `max_rows` | number | não | — | Limite de linhas extraídas |
| `incremental_column` | string | não | — | **Coluna de corte da leitura incremental** (data, timestamp ou sequencial crescente). **PRECISA de índice na origem** |
| `initial_value` | string | não | — | De onde começar na **PRIMEIRA** execução (ex.: `'2026-01-01'`). Vazio = lê tudo |
| `reprocess_window` | number | não | — | **Recuo aplicado ao marco a cada execução**: dias se a coluna for data/hora; unidades se sequencial. Existe para pegar lançamento retroativo |
| `partition_on` | string | não | — | Coluna **numérica** para particionamento paralelo |
| `partition_num` | number | não | `1` | Número de partições paralelas |
| `chunk_size` | number | não | `50000` | Tamanho de cada chunk no streaming |
| `output_field` | string | não | `data` | Nome do campo de saída |
| `read_delivery_mode` | enum | não | `auto` | `auto` / `tunnel` / `edge` — modo de leitura para conexões via relay |

¹ `query` e `table_name` são **alternativos entre si**; o contrato marca só `connection_id` como
obrigatório mas descreve cada um dos dois como "alternativa a" o outro. `[INFERIDO]` Um dos dois
tem de estar preenchido.

---

## Leitura incremental — a marca d'água **é nativa**

`[CONFIRMADO-MCP]` Esta é a descoberta que fechou a lacuna L3. **Não é preciso construir controle
de marca d'água com base interna, tabela de controle ou variável de fluxo** — o nó tem os três
campos abaixo, e o motor guarda o marco entre execuções.

### Os três campos, e o que cada um resolve

| Campo | O que é | O problema que resolve |
|---|---|---|
| `incremental_column` | A **coluna de corte**: data, timestamp ou sequencial crescente | Define *por qual coluna* o motor sabe o que é "novo" |
| `initial_value` | De onde começar na **primeira** execução (ex.: `'2026-01-01'`) | O cold start. **Vazio = lê tudo** — é o pé no chão que evita varrer o histórico inteiro no primeiro run |
| `reprocess_window` | **Recuo aplicado ao marco a cada execução** — dias se a coluna for data/hora, unidades se for sequencial | Lançamento **retroativo**: o registro que entrou com data de ontem depois de o marco já ter passado de ontem |

`[CONFIRMADO-MCP]` Texto literal do contrato sobre o `reprocess_window`:
*"Recuo aplicado ao marco a cada execução: dias, se a coluna for data/hora; unidades, se for
sequencial. **Existe para pegar lançamento retroativo**"*.

### A armadilha de desempenho, declarada

`[CONFIRMADO-MCP]` Literal, sobre o `incremental_column`:

> *"**PRECISA de índice na origem** — sem ele o incremental vira varredura completa e o ganho
> some"*.

> ⚠️ **Consequência operacional para o piloto:** antes de escolher a coluna de corte da tabela de
> pedidos do ERP, é preciso **confirmar com o DBA que existe índice nela**. Uma coluna
> `DATA_ALTERACAO` sem índice transforma a "leitura incremental" numa full table scan a cada
> execução — e o sintoma é lentidão, não erro, então ninguém percebe.
> `[LACUNA]` Não há ferramenta de leitura no MCP que liste índices da origem. O caminho é o
> **playground** (`procedimentos/usar-playground-sql.md`), consultando `USER_INDEXES` /
> `ALL_IND_COLUMNS`. Ver L33.

### Sobreposição de janela = duplicata deliberada

`[INFERIDO]` `reprocess_window` **cria duplicatas de propósito**: se o recuo é de 3 dias, os
registros dos últimos 3 dias voltam a ser lidos em cada execução. Isso é o desenho correto para
não perder lançamento retroativo, mas transfere o problema para o destino — que precisa ser
**idempotente**. Os candidatos confirmados para absorver isso:

- `[CONFIRMADO-MCP]` `bulk_insert` com `unique_columns` — deduplica **antes** do insert.
- `[CONFIRMADO-MCP]` `bulk_insert` com `load_strategy: upsert` + `merge_keys`.
- `[CONFIRMADO-MCP]` `deduplication` (nó de transformação, `ROW_NUMBER() OVER (PARTITION BY)`,
  permite escolher qual duplicata manter).

### O que o motor guarda, e o que não se sabe

`[LACUNA]` **Onde o marco é persistido, e qual é o seu escopo.** O contrato diz que
`incremental_from` é "reservado pelo motor" e que `initial_value` vale "na PRIMEIRA execução",
o que implica estado persistido entre execuções — mas não diz:
- se o escopo é por **nó**, por **fluxo** ou por **conexão**;
- o que acontece quando o fluxo é **duplicado**, **exportado/importado** ou **renomeado**;
- se existe forma de **resetar** o marco (re-executar um backfill) sem recriar o nó;
- se o marco avança quando a execução **falha no meio** (ou seja: perde-se a janela?).

Nenhuma ferramenta de leitura do MCP expõe esse estado. Ver L32, **impacto ALTO** — é a diferença
entre um piloto que pode ser reprocessado e um que não pode.

### Não confundido em aula — nem mencionado

`[VÍDEO]` A aula que apresenta este nó descreve a configuração como praticamente inexistente:
*"Então não tem nenhuma, nenhuma configuração"* (`m3p2b:101`), citando apenas `chunk_size`
(*"para esse caso aqui ele tem bem pouca relevância"*, `m3p2b:103`) e um limite de linhas
(`m3p2b:108`). **A leitura incremental não aparece em nenhuma aula do material.** O recurso existe
só no contrato do MCP. Ver L21.

---

## Particionamento paralelo

`[CONFIRMADO-MCP]` `partition_on` (coluna **numérica**) + `partition_num` (padrão `1`, ou seja
**desligado por padrão**). O motor divide o intervalo da coluna em N faixas e lê em paralelo,
usando internamente os binds reservados **`shift_p_lo` e `shift_p_hi`** — nomes que o nó recusa
como parâmetro do usuário.

`[INFERIDO]` Combinar `partition_on` com `incremental_column` é o caminho para um **backfill
inicial rápido** (ler o histórico particionado por `ID_PEDIDO`) antes de o fluxo cair no regime
incremental do dia a dia. `[LACUNA]` O contrato não diz se as duas coisas podem coexistir na mesma
execução nem como o `WHERE` combinado é montado. Ver L34.

## Streaming e limites de linha

`[CONFIRMADO-MCP]` `chunk_size` (padrão 50.000) é o tamanho de cada chunk no streaming;
`max_rows` é o teto de linhas extraídas.

`[VÍDEO]` Em aula, sobre o limite: *"se você quer trazer um limite de linhas e tudo mais (…) você
poderia muito bem vir aqui e colocar um ROWNUM"* — `m3p2b:108`. Ou seja: o instrutor trata
`ROWNUM` no SQL e o campo de limite como caminhos equivalentes.

> ⚠️ `[UI-OBSERVADA]` **Teto de 500 linhas no modo Teste, independente da query.** Este é o fato
> mais importante desta seção e não está no contrato do MCP:
>
> *"quando tá em teste ele nunca vai trazer mais do que 500 dados. Então mesmo que você rode, sei
> lá, você tem um fluxo de inserir nota fiscal no sistema, se você deixar como teste, ele vai
> sempre inserir só 500, por mais que a consulta tenha 50.000 registros"* — `m2p5:105`
>
> O toggle **Teste / Produção** fica no canto superior direito do nó (`m2p5:104`). Na aula, um
> `SELECT` com `TOP 100` removido de propósito ainda voltou exatamente 500 linhas (`m2p5:97-102`).
> Justificativa narrada: *"a gente sempre prioriza a velocidade"* (`m2p5:103`).
>
> **Consequência para o piloto:** validar um número de margem contra 500 pedidos não prova nada
> sobre o volume real, e um fluxo esquecido em Teste **silenciosamente processa só 500 linhas**.
> `[LACUNA]` O valor 500 é configurável? Vale para todos os nós ou só para os de banco? Ver L35.

## Consultas salvas (playground → nó)

`[UI-OBSERVADA]` O nó tem um campo **`SQL PERSONALIZADO`** (`m2p5:96`) e, ao escolher a conexão,
*"as queries salvas, elas aparecem aqui"* (`m2p5:92`). É o elo entre o copiloto do playground e o
fluxo — ver `procedimentos/usar-playground-sql.md`.

`[LACUNA]` **O contrato do MCP não tem parâmetro de "consulta salva".** `describe_node` só expõe
`query` e `table_name`. Não se sabe se a UI resolve a consulta salva para texto em `query` no
momento de salvar o nó, ou se existe um campo de referência que o MCP não devolve. Ver L36.
Agrava: na própria aula **o salvamento da consulta falhou com erro** (`m2p5:84-86`).

## Entradas esperadas

Nenhuma obrigatória — é nó de **entrada**. `[VÍDEO]` Em aula é sempre ligado logo depois de um
gatilho `manual` (`m2p5:91`, `m3p2b:99-100`).

`[CONFIRMADO-MCP]` Os `parameters` podem vir de **valor fixo, variável do fluxo ou campo do passo
anterior**, então o nó **aceita** upstream quando parametrizado.

## Saídas produzidas

`[CONFIRMADO-MCP]` Um dataset tabular no campo `output_field` (padrão `data`), materializado em
DuckDB.

`[UI-OBSERVADA]` Visível na aba **Output** do painel do nó, em formato de tabela (`m3p2b:104`).

## Erros comuns

`[CONFIRMADO-MCP]` Placeholder em comentário SQL vira bind obrigatório — falha pedindo um valor
"que ninguém sabe de onde saiu".

`[CONFIRMADO-MCP]` Parâmetro que resolve para `None` → **recusado antes de tocar o banco**.

`[CONFIRMADO-MCP]` Usar `incremental_from`, `shift_p_lo` ou `shift_p_hi` como nome de parâmetro →
recusado.

`[CONFIRMADO-MCP]` `incremental_column` sem índice → **não dá erro**, degrada para varredura
completa. É o pior tipo de falha: silenciosa.

`[VÍDEO]` Conexão não informada → *"Connection string obrigatória"* (`m3p2b:115` — o erro ocorreu
no `sql_script`, mas `connection_id` é obrigatório igualmente aqui).

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Extração simples com filtro fixo
{"query": "SELECT * FROM clientes WHERE ativo = 1", "connection_id": "<uuid>"}

// Filtro por valor que varia conforme o fluxo
{"query": "SELECT * FROM pedidos WHERE cnpj = :cnpj",
 "connection_id": "<uuid>",
 "parameters": {"cnpj": {"mode": "dynamic", "template": "{{vars.CNPJ_CLIENTE}}"}}}
```

`[INFERIDO]` Forma provável do nó incremental do piloto — **não confirmada, ver L31**:

```json
{"connection_id": "<uuid oracle>",
 "table_name": "PEDIDO",
 "incremental_column": "DATA_ALTERACAO",
 "initial_value": "2026-08-01",
 "reprocess_window": 3,
 "chunk_size": 50000}
```

## Exportação para SQL / Python

`[CONFIRMADO-DOC]` ⚠️ **A documentação se contradiz sobre este nó.**
`guias-de-uso/exportar-e-importar.md § Cobertura V1` lista `sql_database` entre os **16 tipos
exportáveis** ("Entradas: `sql_database`, `inline_data`") **e** lista `extractNode` entre os tipos
que devolvem **HTTP 422** ("I/O externa: … `extractNode`, `sql_script`").

`[CONFIRMADO-MCP]` `describe_node('sql_database')` declara `extractNode` como **alias** — logo, os
dois nomes são o **mesmo nó**, e a doc o coloca nas duas listas ao mesmo tempo. Registrado como
divergência; **não escolhi um lado**. Ver `divergencias.md`.

## Observações para o piloto de margem

1. **É aqui que o piloto começa.** `incremental_column` + `initial_value` + `reprocess_window`
   substituem inteiramente a tabela de controle que o plano §5.1 previa construir à mão. É a maior
   economia de complexidade descoberta no projeto.
2. **Antes de escrever uma linha de fluxo, confirmar o índice.** Sem índice na coluna de corte, o
   incremental é teatro. O playground é o lugar para checar (`USER_INDEXES`).
3. **`reprocess_window` obriga idempotência a jusante.** Escolher o recuo é escolher quantas
   duplicatas o destino vai receber. Recuo de 3 dias num fluxo diário = cada pedido é reprocessado
   3 vezes. Combine com `bulk_insert`/`unique_columns` ou `deduplication`.
4. **O modo Teste mente sobre volume (500 linhas).** Qualquer validação de "quantos pedidos estão
   fora de faixa" feita em Teste é uma amostra, não um número. E um fluxo publicado que ficou em
   Teste processa 500 e reporta sucesso.
5. **`read_only` é o que permite usar este nó livremente na fase de descoberta.** É o único nó de
   banco do lote que não exige o checklist de pré-produção.
6. **Não confundir com `sql_script`.** Para o piloto: `sql_database` para **ler** o volume de
   pedidos (streaming, particionamento, incremental); `sql_script` só para o **UPDATE** de preço,
   que é o que ele faz e este nó não faz.
7. `[LACUNA]` **O estado do marco (L32) é o risco aberto mais grave deste nó.** Sem saber como
   resetar o marco, um backfill mal disparado ou uma execução interrompida podem exigir recriar o
   nó — perdendo a rastreabilidade do fluxo publicado.
