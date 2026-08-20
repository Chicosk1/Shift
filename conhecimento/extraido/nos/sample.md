# Nó `sample` — Amostra (Sample)

| | |
|---|---|
| **Tipo (MCP)** | `sample` — nenhum alias declarado em `describe_node('sample')` |
| **Rótulo na interface** | Amostragem `[UI-OBSERVADA]` — `m3-3G:3` |
| **Categoria** | `transform` (grupo *Transformação*) `[CONFIRMADO-MCP]` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | `transform` com step `sample` `[CONFIRMADO-DOC]` (`consolidacao-de-nos-de-transformacao.md`) — mas o nó **`transform` NÃO EXISTE** no catálogo: `describe_node('transform')` responde *"Tipo de nó 'transform' não encontrado no catálogo"*. `[CONFIRMADO-MCP]` Ver `divergencias.md` D-MCP-3, já registrada em `mapper.md`. **Na prática, `sample` não tem sucessor disponível** |

## O que faz

`[CONFIRMADO-MCP]` Amostra o dataset upstream usando `SAMPLE` ou `LIMIT` do DuckDB, em três modos:
- **`first_n`** — `LIMIT`
- **`random`** — reservoir com seed
- **`percent`** — Bernoulli
— `describe_node('sample')`

`[CONFIRMADO-DOC]` O SQL exato de cada modo:
- `first_n` → `LIMIT n`
- `random` → `USING SAMPLE reservoir(n ROWS) REPEATABLE(seed)`
- `percent` → `USING SAMPLE p PERCENT (BERNOULLI)`
— `guias-de-uso/nos/sample-(amostra).md`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('sample')`
- Testar um fluxo com um subconjunto dos dados de produção.
- Selecionar N linhas aleatórias de forma reproduzível com seed.
- Amostrar uma porcentagem do dataset para análise rápida.

> `[VÍDEO]` Ressalva importante da aula sobre o escopo do nó: *"na parte de migração, ele vai ser um
> pouco mais raro"*. O uso previsto é **análise estatística**, quando não basta "pegar as primeiras
> tantas linhas". — `m3-3G:4-5`

`[VÍDEO]` Uso concreto citado: testar o fluxo com dados diferentes sem rodar a base inteira —
*"roda com um seed, depois roda com outro, e ele vai gerando amostragens aleatórias"*. — `m3-3G:23-25`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('sample')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mode` | enum | **sim** | `first_n`, `random` ou `percent` |
| `n` | number | condicional | Nº de linhas — obrigatório em `first_n` e `random` |
| `percent` | number | condicional | Percentual entre 0 e 100 — obrigatório em `percent` |
| `seed` | number | não | Seed de reprodutibilidade no modo `random` (**padrão 42**) |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

### Rótulos correspondentes na tela

`[UI-OBSERVADA]` — `m3-3G`

| Campo na tela | Parâmetro |
|---|---|
| **MODO** — "Aleatória (com seed)", "Percentual" (`m3-3G:13,47`) | `mode` |
| **QUANTIDADE DE LINHAS (N)** (`m3-3G:9`) | `n` |
| **SEED (REPRODUTIBILIDADE)** (`m3-3G:15`) | `seed` |
| **PERCENTUAL (%)** (`m3-3G:49`) | `percent` |

> ⚠️ **DIVERGÊNCIA de contrato — duas fontes, dois padrões.**
>
> | Campo | `describe_node('sample')` `[CONFIRMADO-MCP]` | `guias-de-uso/nos/sample-(amostra).md` `[CONFIRMADO-DOC]` |
> |---|---|---|
> | `mode` | **obrigatório**, sem padrão | **não obrigatório**, padrão `first_n` |
> | `seed` | padrão **`42`** | padrão **"aleatório"**; sem seed explícito emite o warning `non_reproducible_sample` |
>
> As duas versões registradas. `[LACUNA]` Falta descobrir qual vale no executor. **Impacto real:**
> se o padrão for `42`, uma amostra sem seed é reprodutível por acidente; se for aleatório, duas
> execuções do mesmo fluxo devolvem conjuntos diferentes — e o warning existe justamente para
> avisar disso. Em fluxo publicado, **declarar `seed` explicitamente** resolve os dois casos.

## Entradas esperadas

Um dataset tabular upstream. `[VÍDEO]` Na aula o `sample` vem depois de `Ler Base Interna` →
`Mapeamento`. — `m3-3G:1`

## Saídas produzidas

`[CONFIRMADO-MCP]` O subconjunto amostrado em `output_field`.

`[CONFIRMADO-DOC]` `output_summary` com `row_count_in` (linhas no upstream), `row_count_out` (linhas
amostradas) e `warnings` — incluindo `non_reproducible_sample` no modo `random` sem `seed` explícito:
*"Em workflows publicados, defina o seed para garantir reprodutibilidade entre execuções."*
— `guias-de-uso/nos/sample-(amostra).md`

### Como o `seed` se comporta

`[UI-OBSERVADA]` Texto de ajuda da própria tela, citado da aula:
> *"o seed é uma semente numérica usada pelo sorteador aleatório. Como o computador gera aleatórios
> a partir dessa semente, fixar o número faz com que o mesmo conjunto de linhas sempre seja
> escolhido. Mesma execução, mesma amostra."*
> *"Quando manter o mesmo? Ao depurar um fluxo, comparar resultados entre execuções ou reproduzir
> relatórios reprodutíveis."* *"E quando trocar? Quando quiser uma amostra diferente."*
> — `m3-3G:17-22`

`[VÍDEO]` Verificado ao vivo: com `seed = 42` e `n = 10`, três execuções seguidas devolveram as
mesmas linhas (Ana Silva, Helena Gomes). Trocando para `35`, o conjunto mudou; voltando para `42`,
voltou ao original. — `m3-3G:27-44`

### Como o `percent` se comporta — não é regra de três

`[UI-OBSERVADA]` Texto de ajuda da tela, citado da aula:
> *"Cada linha do dataset tem essa probabilidade em percentual de entrar na amostra. Por isso, o
> número final pode variar levemente entre execuções. Um dataset de 1000 linhas com 10% pode
> retornar 98 ou 105, por exemplo. Use quando o tamanho exato não importa, mas você quer manter a
> proporção do dataset. Para um número fixo, use aleatória com seed."* — `m3-3G:52-54`

`[VÍDEO]` Observado: 301 linhas com 10% devolveu **31** linhas. — `m3-3G:50`
`[CONFIRMADO-DOC]` Confirma a mecânica: *"`percent` (Bernoulli) decide por linha — variância no
tamanho final."* — `guias-de-uso/nos/sample-(amostra).md`

## Erros comuns

`[CONFIRMADO-DOC]` Guardrails — `guias-de-uso/nos/sample-(amostra).md`:
- `n` negativo → erro.
- `percent` fora de (0, 100] → erro.
- Modo desconhecido → erro com lista de modos válidos.

`[VÍDEO]` Esperar que `percent` devolva um número exato de linhas. A aula insiste no ponto:
*"não é um percentual... tem mil, então vai trazer 10%... Não, ele pode ser 98 ou 105"*. — `m3-3G:57-59`

`[CONFIRMADO-DOC]` Performance: `random` (reservoir) é **O(linhas)** — precisa ler tudo para decidir.
Só `first_n` é O(n), porque o DuckDB para ao atingir o limite. Amostrar aleatoriamente de uma base
grande **não** economiza leitura. — `guias-de-uso/nos/sample-(amostra).md`

## Exemplos

`[CONFIRMADO-MCP]` — `describe_node('sample')`:

```json
// 500 linhas aleatórias, reproduzível
{"mode": "random", "n": 500, "seed": 42}
```

`[CONFIRMADO-DOC]` — `consolidacao-de-nos-de-transformacao.md`:

```json
{"type": "sample", "mode": "first_n", "n": 500, "output_field": "data"}
```

## Observação para o piloto

1. **É a ferramenta de `dry-run` mais honesta que a plataforma tem.** Antes de deixar um fluxo
   gravar preço em produção, rodar a lógica de margem sobre `sample` com `mode: random` e `seed`
   fixo dá um conjunto **estável** para comparar execução a execução — é exatamente o caso de uso
   "depurar um fluxo, comparar resultados entre execuções" do texto de ajuda.
2. **Sempre declarar o `seed` explicitamente**, dada a divergência de padrão acima. Sem isso não há
   garantia de que "rodei de novo e deu diferente" seja bug em vez de amostra nova.
3. **Nunca `percent` para validação de contagem.** Se a conferência é "essas 100 linhas de teste",
   `percent` devolve 98 ou 105 e a conferência quebra sem motivo. `mode: random` + `n` fixo.
4. **Cuidado com `sample` num fluxo que escreve.** O nó é `read_only`, mas colocá-lo antes de um nó
   de gravação significa gravar **parte** dos dados. Como recurso de teste é ótimo; esquecido no
   fluxo publicado, é meia carga silenciosa. `[INFERIDO]` — nenhuma fonte alerta sobre isso.
