# Nó `stat_cohort` — Coorte / Retenção

| | |
|---|---|
| **Tipo (MCP)** | `stat_cohort` |
| **Rótulo na interface** | Coorte / Retenção |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `coorte`, `cohort`, `retencao`, `retention` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

## O que faz

`[CONFIRMADO-MCP]` "Agrupa clientes pela data da primeira compra e mede a retenção ao longo dos
períodos seguintes. Saída em formato longo (coorte, período, % retido). Mostra se a base retém quem
entra." — `describe_node('stat_cohort')`

Mecanismo declarado:

1. Determina a coorte (**primeiro período**) de cada cliente.
2. Calcula o **offset de período** de cada atividade em relação à coorte.
3. **Conta clientes únicos por (coorte, offset) e divide pelo tamanho da coorte.**

— `describe_node('stat_cohort')`, seção *Transforms disponíveis*

## Quando usar

`[CONFIRMADO-MCP]`
- Medir retenção de clientes por safra de aquisição.
- Ver se clientes novos continuam comprando após N meses.
- Direcionar reativação para coortes que caíram.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_cohort')`. É o nó com **menos** parâmetros da categoria.

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `customer_column` | string | **sim** | — | Coluna que identifica o cliente (ex.: `cliente_cnpj`) |
| `date_column` | string | **sim** | — | Coluna de data da transação |
| `granularity` | enum | não | `month` | `month` / `quarter`. "Use `quarter` para B2B irregular" |
| `output_field` | string | não | `data` | Nome do campo de saída |

Não há `min_samples`, não há filtro de coorte mínima, não há teto de períodos.

## Entradas esperadas

`[CONFIRMADO-MCP]` Transações — uma linha por atividade de cliente, com identificador e data.

`[CONFIRMADO-MCP]` **A coorte é definida pelo primeiro período presente no dataset**, não pela
primeira compra real do cliente na vida. Consequência direta e importante: se a extração começa em
janeiro de 2026, todo cliente antigo que comprou em janeiro é classificado como coorte de janeiro de
2026 — a base parece ter adquirido milhares de clientes num mês só.

## Saídas produzidas

`[CONFIRMADO-MCP]` **Formato longo**: uma linha por (coorte, período, % retido). Não é a matriz
triangular pronta — quem quiser o triângulo visual precisa pivotar depois.

`[LACUNA]` O contrato **não nomeia** as colunas exatas de saída, nem diz se o offset 0 (o próprio
período de aquisição, sempre 100%) vem incluído.

## Erros comuns / armadilhas

Este nó **não tem seção "Armadilhas conhecidas"** no contrato. As consequências abaixo derivam do
mecanismo declarado.

`[CONFIRMADO-MCP]` **Coorte truncada à esquerda — erro silencioso.** A coorte sai do "primeiro
período" *no dado recebido*. Um recorte de janela (filtro de data upstream, extração incremental)
faz clientes veteranos aparecerem como novos. O resultado é uma matriz de retenção plausível e
enganosa, sem erro. Para ler retenção de verdade, a entrada tem de conter o histórico inteiro do
cliente, não uma janela.

`[CONFIRMADO-MCP]` **A última coorte tem poucos períodos observados.** Como o offset máximo depende
de quanto tempo passou desde a coorte, as safras recentes têm a linha curta. Isso é da natureza da
análise, mas quem tirar média de retenção por offset misturando coortes com maturidade diferente
compara coisas diferentes. O nó não sinaliza nada.

`[CONFIRMADO-MCP]` **Coorte de tamanho 1 ou 2 produz retenção de 0% ou 100%.** Sem `min_samples`, o
nó divide por qualquer denominador. Coortes minúsculas viram ruído com cara de padrão.

`[LACUNA]` O contrato **não declara** o que conta como "atividade retida": qualquer linha no
período? valor acima de zero? Nem o que faz com cliente que tem data nula.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// "Como montar a matriz de retenção por coorte mensal?"
{"customer_column": "cliente_cnpj", "date_column": "data_venda", "granularity": "month"}
```

## Observações

1. **Granularidade só `month` ou `quarter`.** Não há `week` nem `day`. Para supermercado, onde a
   frequência de compra é semanal, **mensal é grosseiro**: um cliente que comprava toda semana e
   passou a comprar uma vez no mês continua "retido" 100% na leitura mensal. A queda de frequência
   é invisível aqui — quem quer medir isso precisa do `stat_rfm` (eixo F) ou do `stat_transition`
   sobre faixa de frequência.
2. **É o par do `stat_rfm`:** RFM diz quem está em risco agora, coorte diz se o problema é
   sistêmico na aquisição.
3. `[LACUNA]` Mesmo problema de identificação de cliente que o `stat_rfm`: exige
   `customer_column` populada. Cupom anônimo de supermercado não entra na conta, e o nó não avisa
   que 70% do faturamento ficou fora.
