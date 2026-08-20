# Nó `stat_market_basket` — Cesta de Compras (Market Basket)

| | |
|---|---|
| **Tipo (MCP)** | `stat_market_basket` |
| **Rótulo na interface** | Cesta de Compras (Market Basket) |
| **Categoria** | `statistics` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases** | `cesta`, `market basket`, `apriori`, `cross-sell`, `regras de associacao` |
| **Cobertura no acervo** | **Só MCP** — nenhuma aula, nenhum documento (lacuna L19) |

## O que faz

`[CONFIRMADO-MCP]` "Descobre **pares** de itens comprados juntos com support, confidence e lift.
Base para kits, layout de loja e recomendação (ex: cimento+areia, tinta+rolo)."
— `describe_node('stat_market_basket')`

Mecanismo declarado:

1. **Deduplica itens por transação** e cruza os itens da mesma transação (**self-join**).
2. Calcula suporte, confiança e lift de cada **par direcional X→Y**.
3. Filtra por `min_support` / `min_confidence` e **ordena por lift**.

— `describe_node('stat_market_basket')`, seção *Transforms disponíveis*

**Só pares.** O contrato diz "pares de itens" na descrição e "cada par direcional X→Y" no transform.
Apesar do alias `apriori`, não há indício de itemset de tamanho 3 ou mais — e nenhum parâmetro para
pedir isso.

## Quando usar

`[CONFIRMADO-MCP]`
- Descobrir o que é vendido em conjunto.
- Montar kits e combos ou sugerir cross-sell.
- Organizar layout de loja / gôndola por afinidade.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('stat_market_basket')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `transaction_column` | string | **sim** | — | Coluna que identifica a transação/nota (agrupa os itens de uma compra) |
| `item_column` | string | **sim** | — | Coluna do item/produto |
| `min_support` | number | não | `0.01` (=1%) | Suporte mínimo do par, **fração de transações** |
| `min_confidence` | number | não | `0.1` | Confiança mínima da regra, 0–1 |
| `max_rules` | number | não | `200` | Nº máximo de regras retornadas, **ordenadas por lift** |
| `output_field` | string | não | `data` | Nome do campo de saída |

## Entradas esperadas

`[CONFIRMADO-MCP]` Itens de venda em formato longo: uma linha por item de nota, com a chave da
transação e o identificador do produto. Repetição do mesmo item na mesma nota é tratada — o nó
**deduplica itens por transação** antes de cruzar.

`[LACUNA]` O contrato **não declara** o que acontece com transação de item único (não gera par, mas
entra no denominador do suporte? presumivelmente sim, e isso importa muito no varejo). Não afirmado.

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma linha por par direcional X→Y, com **support**, **confidence** e **lift**,
ordenadas por lift decrescente, limitadas a `max_rules`.

`[LACUNA]` O contrato não nomeia as colunas de saída nem declara as fórmulas exatas de support,
confidence e lift (as definições clássicas são conhecidas, mas o contrato não as escreve — em
particular, se `support` é do par ou do antecedente).

## Erros comuns / armadilhas

Este nó **não tem seção "Armadilhas conhecidas"** no contrato. As abaixo derivam do mecanismo
declarado, e a primeira é séria para supermercado.

`[CONFIRMADO-MCP]` **`min_support` de 1% é alto para supermercado — e o efeito é silencioso.** Com
`transaction_column` = cupom e 500 mil cupons, 1% exige que o par apareça em 5.000 cupons. Num
sortimento de 20 mil SKUs, quase nenhum par de produtos específicos alcança isso; só as combinações
de itens de altíssimo giro (pão + leite) sobrevivem — e essas são exatamente as regras que ninguém
precisa descobrir. O nó devolve poucas regras óbvias, ou nenhuma, **sem erro e sem aviso** de que o
suporte está cortando tudo. Solução: baixar `min_support`, ou rodar sobre **categoria** em vez de SKU
(basta apontar `item_column` para a coluna de categoria).

`[CONFIRMADO-MCP]` **Lift alto com suporte baixo é ruído.** A ordenação é **por lift**, e lift
premia par raro: dois itens que apareceram juntos 3 vezes e sozinhos nunca têm lift enorme. Como as
`max_rules` são preenchidas por lift, baixar `min_support` demais enche o top-200 de coincidências.
O contrato não avisa — a defesa é ler a coluna de support junto com o lift, sempre.

`[CONFIRMADO-MCP]` **`max_rules` = 200 corta silenciosamente.** Igual ao `max_series` do
`stat_forecast`: o excedente desaparece sem sinalização.

`[CONFIRMADO-MCP]` **Self-join tem custo quadrático no tamanho da cesta.** O contrato declara o
mecanismo ("cruza os itens da mesma transação (self-join)") mas **não** declara o custo — diferente
do `stat_anomaly`, que quantifica o seu. `[LACUNA]` Uma nota de atacado com 500 itens gera ~125 mil
pares só ela; qual o limite prático, o contrato não diz.

`[CONFIRMADO-MCP]` **Regras direcionais aparecem em dobro.** X→Y e Y→X são pares distintos com
confidence diferente e o **mesmo** lift (lift é simétrico). Quem contar regras vai contar cada
afinidade duas vezes.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// "Quais produtos são comprados junto?"
{"transaction_column": "nota", "item_column": "produto", "min_support": 0.01,
 "min_confidence": 0.1}
```

## Observações

1. **Aplicável a supermercado sem nenhuma adaptação de dado** — é o único nó da categoria que roda
   direto sobre item de cupom, sem exigir cliente identificado, snapshot por período ou desfecho
   observado. Basta a chave do cupom e o produto. Isso o torna o nó de menor atrito do lote.
2. **Rodar por categoria primeiro, por SKU depois** é a estratégia que o `min_support` impõe na
   prática. Não está no contrato; é consequência dele.
3. `[LACUNA]` Nada sobre janela temporal. Afinidade muda com sazonalidade (churrasco no verão), e o
   nó não tem `date_column`. Recortar o período é responsabilidade do upstream.
4. **Não confundir com recomendação personalizada.** O nó devolve afinidade agregada de produto, não
   preferência por cliente — não há `customer_column`.
