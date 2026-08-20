# Nó `stat_transition` — Matriz de Transição (Roll Rate)

| | |
|---|---|
| **Tipo (MCP)** | `stat_transition` |
| **Rótulo na interface** | Matriz de Transição |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `transicao`, `matriz de transicao`, `roll rate`, `rollrate`, `markov`, `migracao`, `aging` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

> **A armadilha mais grave de toda a categoria está neste nó.** Está declarada **duas vezes** no
> contrato — na descrição e na lista de armadilhas — o que é único no lote. Ver abaixo.

## O que faz

`[CONFIRMADO-MCP]` "Mede no histórico a probabilidade de uma entidade migrar de um estado para outro
entre períodos consecutivos (roll rate). Saída em formato longo (passo, estado de origem, estado de
destino, probabilidade)." — `describe_node('stat_transition')`

E, na mesma descrição, o requisito de entrada em maiúsculas:

> A ENTRADA TEM DE SER POSIÇÃO POR PERÍODO (snapshot): uma linha por entidade e por período, com o
> estado dela naquele período. Se o upstream mandar EVENTOS (uma linha por ocorrência), o nó mede a
> transição entre ocorrências consecutivas em vez de entre períodos e devolve um número plausível e
> errado, sem erro nenhum.

Mecanismo declarado:

1. Reamostra a posição por período pegando o **último estado observado** dentro dele.
2. Casa cada período com o **período seguinte da mesma entidade** e conta os pares.
3. Divide pelo total de saídas da origem para obter a probabilidade.
4. **Eleva a matriz a N passos** quando `projection_steps > 0`.

— `describe_node('stat_transition')`, seção *Transforms disponíveis*

## Quando usar

`[CONFIRMADO-MCP]`
- Estimar quanto do vencido de hoje tende a virar perda (roll rate de aging).
- Medir como pedidos, chamados ou oportunidades migram entre status ao longo do tempo.
- Ver a chance de um cliente mudar de faixa (**curva ABC**, faixa de compra) no próximo período.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_transition')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `entity_column` | string | **sim** | — | Coluna que identifica a entidade (ex.: `cliente_cnpj`, `titulo_id`) |
| `date_column` | string | **sim** | — | Coluna da data **da posição** |
| `state_column` | string | condicional | — | Coluna que **já contém** o estado. Use esta **OU** `value_column`+`bucket_edges`, **nunca as duas** |
| `value_column` | string | condicional | — | Alternativa: coluna numérica a ser faixada (ex.: `dias_atraso`). **Exige `bucket_edges`** |
| `bucket_edges` | array | condicional | — | Cortes das faixas, ex.: `[0, 30, 60, 90]`. Gera **N+1** faixas (abaixo do 1º corte e acima do último) |
| `bucket_labels` | array | não | — | Rótulos; se informado precisa ter **exatamente `len(bucket_edges)+1`** itens |
| `granularity` | enum | não | `month` | `month` / `quarter` |
| `include_exit_state` | boolean | não | `true` | Entidade que some no período seguinte transita para o estado de saída |
| `exit_label` | string | não | `Saiu` | Rótulo do estado de saída |
| `min_samples` | number | não | `10` | Mínimo de ocorrências **na origem** para publicar a probabilidade |
| `projection_steps` | number | não | `0` | Acrescenta as matrizes de N passos à frente. "Horizonte curto: até 3" |
| `output_field` | string | não | `data` | Nome do campo de saída |

## Entradas esperadas

`[CONFIRMADO-MCP]` **Snapshot, não evento.** Uma linha por (entidade × período), com o estado
naquele período. Esta é a exigência mais dura de toda a categoria de estatística, e a mais fácil de
violar sem perceber — a maior parte das tabelas de ERP é de eventos (movimento, lançamento, item de
nota), não de posição.

`[CONFIRMADO-MCP]` Duas formas mutuamente exclusivas de definir o estado:
- `state_column` — o estado já vem pronto na coluna (status de pedido, classe ABC).
- `value_column` + `bucket_edges` — o nó faixa um número (dias de atraso, valor de compra).

## Saídas produzidas

`[CONFIRMADO-MCP]` **Formato longo**: passo, estado de origem, estado de destino, probabilidade.
Colunas adicionais declaradas nas armadilhas: `ocorrencias`, `total_origem`, `amostra_suficiente`.

`[CONFIRMADO-MCP]` Linhas com `passo > 1` são **projeção**, não observação — e vêm com `ocorrencias`
e `total_origem` **nulos de propósito**.

## Erros comuns / armadilhas

`[CONFIRMADO-MCP]` As sete armadilhas do contrato, literais — `describe_node('stat_transition')`,
seção *Armadilhas conhecidas*:

1. > Exige POSIÇÃO por período (snapshot), não eventos. Com eventos o nó mede a transição entre
   > ocorrências consecutivas e devolve um número plausível e errado — não há erro que avise.

2. > Origem com menos ocorrências que min_samples devolve probabilidade NULL (nunca zero) e
   > amostra_suficiente falso. Um filtro downstream que trate NULL como zero infla a leitura.

3. > Preencher state_column E value_column ao mesmo tempo, ou nenhum dos dois, levanta erro: são
   > caminhos alternativos, não complementares.

4. > value_column sem bucket_edges levanta erro; bucket_labels, quando informado, tem de ter
   > exatamente len(bucket_edges)+1 itens.

5. > Entidade sem estado no período (state_column ou value_column NULL) vira o estado '(sem estado)'
   > em vez de sumir — se sumisse, seria contada como saída e inflaria a probabilidade de saída.

6. > As linhas de projeção (passo > 1) vêm com ocorrencias e total_origem NULOS de propósito: são
   > derivadas, não observadas. Filtre por passo = 1 para ver só o histórico.

7. > A projeção assume matriz estável no horizonte, e trata como absorvente (fica onde está) toda
   > origem sem amostra suficiente — inclusive o estado de saída. Horizonte curto: até 3 passos,
   > teto de 12.

### Hierarquia de gravidade

- **Erra em silêncio:** #1 (eventos em vez de snapshot — a pior de todas), #2 (NULL lido como zero
  downstream), #7 (projeção com matriz instável, e absorção indevida de origens sem amostra).
- **Falha alto, o que é bom:** #3 e #4 — configuração contraditória levanta erro na hora.
- **Comportamento deliberado que confunde na leitura:** #5 e #6.

`[CONFIRMADO-MCP]` Sobre a #1: a frase *"devolve um número plausível e errado, sem erro nenhum"* é a
mais forte de todo o contrato do MCP. **Verificar a granularidade da entrada é pré-requisito de uso
deste nó**, não boa prática opcional.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// "Qual a chance de um título que está entre 31 e 60 dias de atraso piorar no mês que vem?"
{"entity_column": "titulo_id", "date_column": "data_posicao", "value_column": "dias_atraso",
 "bucket_edges": [0, 30, 60, 90], "granularity": "month", "projection_steps": 3}

// "Como os pedidos migram entre status de um mês para o outro?"
{"entity_column": "pedido_id", "date_column": "data_posicao", "state_column": "status",
 "granularity": "month"}
```

## Observações

1. **Como fabricar o snapshot que o nó exige.** O ERP raramente tem tabela de posição mensal. O
   caminho é gerar a posição no upstream — SQL que, para cada período, resolve o estado da entidade
   naquele fechamento — e só então entregar ao nó. `[LACUNA]` Não há nó de estatística que faça isso;
   é trabalho de `sql_script` / transformação, e o contrato deste nó não indica como.
2. **Ligação declarada com `stat_abc`.** O "use quando" cita explicitamente migração de classe ABC.
   Encadeamento: `stat_abc` por mês → snapshot (entidade, mês, classe) → `stat_transition`. É o
   caminho para responder "quais produtos estão saindo da classe A".
3. **Reamostragem pega o último estado do período** — logo, mudanças de ida e volta dentro do mesmo
   mês são invisíveis. Com `granularity: month`, um pedido que passou por 4 status em 20 dias conta
   como um único estado.
4. `[LACUNA]` O contrato não declara quantos períodos de histórico são necessários para a matriz ser
   estável, nem quantas entidades. `min_samples: 10` é por célula de origem, não pela matriz.
