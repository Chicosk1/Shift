# Nó `join` — Junção (Cruzamento) — ⚠️ LEGADO

> ⚠️ `[CONFIRMADO-MCP]` **Nó legado.** Citado literalmente do contrato:
> *"Versão legada — prefira o nó `combine` com mode='join'."* — `describe_node('join')`
>
> Documentação nova deve apontar para **[`combine.md`](combine.md)**. Este arquivo existe porque o
> nó **continua funcionando**, é o que aparece em todo o material de treinamento, e é o que está
> nos fluxos já construídos.

| | |
|---|---|
| **Tipo (MCP)** | `join` — nenhum alias declarado em `describe_node('join')` |
| **Rótulo na interface** | Junção (Join) `[UI-OBSERVADA]` — `m3-3J:26`; o instrutor fala "nó de Junção" — `m3-3J:3` |
| **Categoria** | `transform` (grupo *Transformação*) `[CONFIRMADO-MCP]` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | **`combine` com `config.mode = "join"`** `[CONFIRMADO-MCP]` + `[CONFIRMADO-DOC]` (`consolidacao-de-nos-de-transformacao.md`). O sucessor **existe e está no catálogo** — diferente do caso `transform`, que é documentado mas ausente |

## O que faz

`[CONFIRMADO-MCP]` Cruza dois datasets (`left` e `right`) via SQL JOIN, com suporte a
`inner` / `left` / `right` / `full`. — `describe_node('join')`

> `[VÍDEO]` A aula descreve o nó como um SQL montado por interface: *"é um SQL, né? Só que é um SQL
> aqui que você consegue, através de uma interface, você configurar. Você não precisa entender de
> SQL, mas você tem que entender da lógica"*. — `m3-3J:70,72`
> O instrutor escreve no bloco de notas o equivalente:
> `FROM TABELA 1 / JOIN TABELA 2 / ON TABELA1.COLUNA = TABELA2.COLUNA`. — `m3-3J:33-35,51-52`

### ⚠️ O que o `join` NÃO faz: cross join

`[CONFIRMADO-MCP]` O contrato do `join` lista apenas `inner`, `left`, `right`, `full`.
`[VÍDEO]` A aula confirma, explicitamente: *"A gente tem o Inner, tem o Left, tem o Right, tem o
Full. A gente só não tem o Cross aqui, né, mas a gente tem em outro... outro nó ali."*
— `m3-3J:72`

`[VÍDEO]` Em `m3-3H:41-45` esse "outro nó" aparece: o instrutor **tenta o `Junção (Join)`,
não encontra o cross, remove o nó** e usa o **Combinar** no modo "Cruzar" / "Cross join
cartesianos". Essa é a lacuna funcional concreta que motiva a migração.

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('join')`
- Cruzar dois datasets por colunas em comum.
- Adicionar colunas de uma tabela de referência à tabela principal.

`[VÍDEO]` Caso canônico da aula: uma planilha de **vendas** tem só `funcionario_id`; para o
relatório é preciso o **nome** do funcionário, que vive na planilha de **funcionários**.
— `m3-3J:23-24`

`[VÍDEO]` Regra prática dada em aula: *"a minha tabela principal sempre deve ficar no FROM"* — se a
análise é de vendas, Vendas vai no FROM e Funcionários no JOIN. — `m3-3J:43-44`

`[VÍDEO]` Sobre os tipos: *"geralmente usa-se muito o INNER e o LEFT"*. — `m3-3J:50`

### Encadeamento para 3+ tabelas

`[VÍDEO]` O nó *"sempre vai ser a partir de duas tabelas"* (`m3-3J:25`). Para três fontes, a aula
mostra o padrão: faz a primeira junção, e a **saída dela** entra no FROM da segunda junção.
— `m3-3J:27-32,38`

`[VÍDEO]` Prática recomendada em aula: inserir um nó **Mapeamento** entre as junções para
normalizar nomes antes de encadear — *"antes de fazer a junção, eu gosto de organizar as coisas"*.
No exemplo o nó foi batizado "Organização" e renomeou `nome` → `nome_cliente` e `j_nome` →
`nome_funcionario`. — `m3-3J:102-109`

`[VÍDEO]` Para não poluir o canvas, os nós da cadeia foram **agrupados** ("Agrupar selecionados",
grupo "Grupo de Vendas", com cor e modo "Foco"). — `m3-3J:119-123`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('join')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `join_type` | enum | não | `inner` (padrão), `left`, `right`, `full` |
| `conditions` | array | **sim** | Lista `[{left_column, right_column}]` |
| `columns` | array | não | Colunas a selecionar; se omitido, seleciona tudo |
| `output_field` | string | não | Nome do campo de saída (padrão `data`) |

### Rótulos correspondentes na tela

`[UI-OBSERVADA]` — `m3-3J`

| Campo na tela | Parâmetro |
|---|---|
| entradas **FROM** e **JOIN** (`m3-3J:45`) | os handles `left` / `right` |
| **TIPO DE JUNÇÃO** (`m3-3J:69`) | `join_type` |
| **CHAVES DE JUNÇÃO** (`m3-3J:55`) | `conditions` |
| **COLUNAS DE SAÍDA**, abas "Escolher" e "Avançado" (`m3-3J:57`) | `columns` |

`[VÍDEO]` A aba **"Avançado"** permite editar a seleção à mão e *"adicionar alguma expressão SQL a
mais"*; o instrutor avisa que *"vai ser difícil se você precisar mexer aqui"*. — `m3-3J:56-58`
`[LACUNA]` O contrato declara `columns` como um array simples. Falta descobrir como a expressão SQL
do modo Avançado é persistida — se dentro de `columns` ou em outro campo não exposto pelo
`describe_node`.

## ⚠️ Alias automático `f` / `j` e colisão de nomes

`[UI-OBSERVADA]` Citado da aula, porque muda o nome das colunas de saída sem aviso:

> *"por padrão ele sempre gera um alias pra essas tabelas, que é a que tá no FROM é o alias `f`, e o
> que tá no JOIN o alias `j`."* — `m3-3J:56`
>
> *"quando o nome coincide, ele coloca um jota na frente"* — a coluna `nome` de Funcionários chega
> na saída como **`j_nome`**, ao lado do `nome` que veio de Vendas. — `m3-3J:65-66`

`[VÍDEO]` A aula também menciona uma heurística de chave: ao escolher as colunas, o `id` da tabela
do JOIN **não entra** porque *"ele já tem uma inteligência... não vai entrar o id porque ele já tem
o funcionário_id"*. — `m3-3J:61`

> ⚠️ **DIVERGÊNCIA aula ↔ MCP.** Nada no contrato do `join` (`describe_node('join')`) menciona
> alias `f`/`j`, prefixo automático, resolução de colisão ou supressão da coluna-chave duplicada.
> `columns` é declarado apenas como *"Colunas a selecionar; se omitido, seleciona tudo"*.
> **Registradas as duas fontes.** `[LACUNA]` Falta descobrir, executando: (a) se o prefixo `j_` é
> aplicado pelo executor ou só sugerido pelo editor visual ao gravar `columns`; (b) o que acontece
> com `columns` omitido e colunas homônimas nos dois lados — se sai `j_nome` ou se uma sobrescreve
> a outra. **Impacto direto:** um nó downstream que referencie `nome` pode estar lendo a coluna
> errada.

## Entradas esperadas

`[CONFIRMADO-DOC]` Dois datasets, nos handles **fixos `left` e `right`**.
— `consolidacao-de-nos-de-transformacao.md`
`[UI-OBSERVADA]` Na tela esses handles aparecem como **FROM** (= `left`) e **JOIN** (= `right`).
— `m3-3J:45`

## Saídas produzidas

`[CONFIRMADO-MCP]` O dataset cruzado em `output_field` (padrão `data`).

`[VÍDEO]` Comportamento observado com uma venda de `funcionario_id` nulo:
- **Inner** → a linha órfã **não aparece** (`m3-3J:67`).
- **Left** → a linha órfã **aparece**, com as colunas do lado direito nulas (`m3-3J:69-71`).

## Erros comuns

`[VÍDEO]` Ler a coluna errada por causa do prefixo `j_` — ver a seção de alias acima. A aula
resolve renomeando no Mapeamento imediatamente depois (`j_nome` → `nome_funcionario`). — `m3-3J:105`

`[INFERIDO]` Usar `inner` quando o requisito é "não perder nenhum pedido": linhas sem
correspondência somem sem erro e sem warning. É perda **silenciosa** — o contrário do que uma
automação de preço precisa.

`[LACUNA]` O contrato não documenta nenhum `output_summary`, warning ou contagem de linhas para o
`join` — diferente do `union`, que tem `row_count_in` por handle e `schema_drift`. Falta descobrir
se o `join` emite algo observável.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato — `describe_node('join')`:

```json
{"join_type": "inner", "conditions": [{"left_column": "id_cliente", "right_column": "id"}]}
```

`[VÍDEO]` Da aula, cadeia de duas junções — `m3-3J:94-115`:

```
Vendas (FROM) + Funcionários (JOIN)   → inner, funcionario_id = id
   ↓
Mapeamento "Organização"              (j_nome → nome_funcionario, nome → nome_cliente)
   ↓
Organização (FROM) + Departamentos (JOIN) → left, id_departamento = id
```

`[CONFIRMADO-DOC]` A mesma configuração já em `combine` — `consolidacao-de-nos-de-transformacao.md`:

```json
{"type": "combine", "output_field": "data",
 "config": {"mode": "join", "join_type": "left",
            "conditions": [{"left_column": "cliente_id", "right_column": "id"}],
            "columns": null}}
```

## Observação para o piloto

1. **Não usar `join` em fluxo novo.** `combine` cobre os quatro tipos e acrescenta `cross`; os
   nomes dos campos (`join_type`, `conditions`, `columns`) são idênticos, apenas movidos para
   dentro de `config`. O custo de migração é reescrever o envelope, não a lógica.
2. **A conversão é mecânica:** `{join_type, conditions, columns}` → `{config: {mode: "join",
   join_type, conditions, columns}}`. `[CONFIRMADO-DOC]`
3. **Mas o material de treinamento todo fala `join`.** Qualquer skill que ensine junção precisa
   dizer as duas coisas: o que a pessoa vê na aula (Junção, FROM/JOIN) e o que deve escrever hoje
   (`combine`, `left`/`right`).
