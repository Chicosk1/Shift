# Nó `stat_anomaly` — Detecção de Anomalias

| | |
|---|---|
| **Tipo (MCP)** | `stat_anomaly` |
| **Rótulo na interface** | Detecção de Anomalias |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `outlier`, `anomalia`, `zscore`, `z-score`, `cusum`, `ewma`, `mudanca de patamar`, `deterioracao`, `piora gradual`, `tendencia` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

> **Nó com 8 armadilhas declaradas no contrato** — a maior lista de toda a categoria. Todas
> reproduzidas literalmente abaixo.

## O que faz

`[CONFIRMADO-MCP]` "Sinaliza linhas fora do padrão, global ou por grupo, e responde a **DUAS
perguntas diferentes** conforme o método." — `describe_node('stat_anomaly')`

| Pergunta | Métodos | Para que |
|---|---|---|
| **Valor atípico** — "este ponto é estranho perto dos outros?" | `zscore`, `zscore_robust` (mediana/MAD), `iqr` | Fraude, desconto abusivo, erro de digitação, pico isolado |
| **Mudança de patamar** — "esta série mudou de nível e não voltou?" | `cusum`, `ewma` (exigem `date_column`) | Deterioração **gradual**, "onde cada passo isolado parece aceitável" |

`[CONFIRMADO-MCP]` Aviso central do contrato, citado literalmente:

> A linha de base dos dois é RETROSPECTIVA (média e desvio do grupo inteiro, inclusive do trecho
> depois da mudança) — não é controle online.

`[CONFIRMADO-MCP]` A saída é a mesma nos cinco métodos: `anomalia_score` e `is_anomalia`.

## Quando usar

`[CONFIRMADO-MCP]`
- Detectar descontos fora da curva por vendedor ou categoria.
- Achar preços de compra muito acima do mercado.
- Sinalizar picos ou quedas inexplicáveis de venda.
- Pegar o cliente que ainda paga mas começou a atrasar cada vez mais (`cusum`/`ewma`).
- Acompanhar piora lenta de prazo de entrega, consumo **ou margem** (`cusum`/`ewma`).

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_anomaly')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `metric_column` | string | **sim** | — | Coluna numérica a avaliar (ex.: `percentual_desconto`, `preco_unitario`, `dias_atraso`) |
| `group_by` | array | não | vazio = tabela toda | Colunas para avaliar por grupo. Em `cusum`/`ewma`, define **uma série por grupo** |
| `method` | enum | não | `zscore_robust` | `zscore_robust` / `zscore` / `iqr` / `cusum` / `ewma` |
| `date_column` | string | condicional | — | **Obrigatória em `cusum` e `ewma`**; ignorada nos outros três |
| `threshold` | number | não | ver abaixo | Limiar — **o significado muda com o método** |
| `k` | number | não | `0.5` | **Só `cusum`**: folga em desvios antes de acumular. Maior = menos sensível |
| `lambda` | number | não | `0.2` | **Só `ewma`**: peso do dado recente, 0–1. Maior = reage mais rápido e com mais ruído |
| `min_samples` | number | não | `5` | Mínimo de linhas no grupo para julgar. Abaixo disso `is_anomalia=false` |
| `output_field` | string | não | `data` | Nome do campo de saída |

### `threshold` — mesmo campo, cinco significados

`[CONFIRMADO-MCP]`

| Método | O que `threshold` significa | Padrão / recomendado |
|---|---|---|
| `zscore`, `zscore_robust` | Nº de desvios | `3.0` |
| `iqr` | Multiplicador do IQR | "use ~1.5" |
| `cusum` | O parâmetro **`h`** | `4.0` |
| `ewma` | O parâmetro **`L`** | `3.0` |

## Entradas esperadas

`[CONFIRMADO-MCP]` Dataset tabular com a coluna numérica da métrica. Para `cusum`/`ewma`,
adicionalmente uma coluna de data que ordene a série — o nó "ordena cada série pela data e acumula
os desvios padronizados".

## Saídas produzidas

`[CONFIRMADO-MCP]` Duas colunas, iguais para todos os métodos:

- `anomalia_score` — **o sinal carrega a direção**: positivo = subiu, negativo = caiu.
- `is_anomalia` — booleano, marcado "quando o score/regra ultrapassa o limiar **e o grupo tem
  amostra suficiente**".

## Erros comuns / armadilhas

`[CONFIRMADO-MCP]` As oito armadilhas do contrato, literais — `describe_node('stat_anomaly')`,
seção *Armadilhas conhecidas*:

1. > A linha de base de cusum/ewma é RETROSPECTIVA: média e desvio saem do grupo inteiro, inclusive
   > do trecho DEPOIS da mudança. Isso não é controle online — se metade da série está num patamar
   > e metade noutro, as duas metades aparecem afastadas da média e as duas podem ser sinalizadas.
   > Para monitoramento, recorte a janela antes do nó.

2. > cusum e ewma EXIGEM `date_column`; sem ela o nó levanta erro. Os três métodos de valor atípico
   > ignoram esse campo.

3. > O `threshold` é o MESMO campo para os cinco métodos, mas o significado muda: desvios no
   > z-score, multiplicador do IQR, 'h' no cusum, 'L' no ewma. Trocar de método sem revisar o
   > limiar produz sensibilidade muito diferente da esperada.

4. > cusum e ewma têm ATRASO por construção: a mudança só é sinalizada depois de alguns pontos,
   > porque é o acúmulo que denuncia. Um degrau no último ponto da série pode não aparecer.

5. > A direção da mudança vai no SINAL de `anomalia_score` (positivo = subiu, negativo = caiu) —
   > não há coluna nova. Um filtro `anomalia_score > 0` é o que separa piora de melhora quando a
   > métrica é 'quanto maior, pior'.

6. > Grupo com desvio-padrão zero (métrica constante) tem desvio padronizado indefinido: o nó trata
   > como zero, ou seja, não sinaliza nada nesse grupo em vez de estourar.

7. > Linha com métrica NULA entra na série como desvio zero, não interrompe o acúmulo — se sumisse,
   > a série mudaria de comprimento e o ponto seguinte herdaria o acúmulo errado.

8. > A recursão de cusum/ewma custa um passo por ponto da SÉRIE MAIS LONGA (não por linha da
   > tabela): 50 mil clientes × 24 meses são 24 passos. Uma série única de milhões de pontos sem
   > `group_by` é o caso que fica caro.

### As que enganam sem erro

Das oito, **quatro devolvem resultado errado ou enganoso sem levantar erro**: a #1 (base
retrospectiva sinaliza as duas metades), a #3 (limiar reaproveitado entre métodos), a #6 (grupo
constante não sinaliza nada — silêncio que parece "tudo bem") e a #4 (degrau no fim da série não
aparece). A #2 é a única que **falha alto**.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// "Como achar descontos anômalos por categoria de produto?"
{"metric_column": "percentual_desconto", "group_by": ["categoria_produto"],
 "method": "zscore_robust", "threshold": 3.0}

// "Quais clientes começaram a atrasar cada vez mais, mesmo ainda pagando?"
{"metric_column": "dias_atraso", "group_by": ["cliente_id"], "date_column": "data_posicao",
 "method": "cusum", "k": 0.5, "threshold": 4.0}
```

## Observações

1. **`zscore_robust` é o padrão, e isso é uma escolha de projeto favorável ao varejo.** Mediana/MAD
   resiste a outlier — e é justamente o outlier que se está procurando. Usar `zscore` puro numa
   base com fraude faz a própria fraude inflar a média e o desvio, mascarando-se.
2. `[LACUNA]` O contrato **não declara a fórmula exata** de `anomalia_score` em nenhum dos cinco
   métodos, nem qual estatística de escala o `zscore_robust` usa como divisor (MAD bruto ou MAD
   escalado por 1,4826). Comparar o score entre métodos é, portanto, indefinido.
3. `[LACUNA]` Não há **volume mínimo recomendado** além do `min_samples=5` por grupo — e 5 é um
   piso de execução, não um piso de validade estatística. O contrato não sugere quanto é suficiente.
4. **Único nó da categoria que cita margem no "use quando"** (*"piora lenta de prazo de entrega,
   consumo ou margem"*) — mas como métrica genérica a monitorar, sem dizer de onde a margem vem.
