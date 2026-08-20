# Nó `truncate_table` — Truncar/Deletar Tabela / "Limpar Tabela"

| | |
|---|---|
| **Tipo (MCP)** | `truncate_table` |
| **Rótulo no contrato do MCP** | Truncar/Deletar Tabela |
| **Rótulo na interface (aula)** | **Limpar Tabela** — `m3p2b:97,153-156` `[UI-OBSERVADA]` |
| **Categoria** | `database` (grupo *Banco de Dados*) `[CONFIRMADO-MCP]` |
| **Risco** | **`write`** `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

> **Nó de escrita, e o mais destrutivo do catálogo.** O nome "Limpar Tabela" é eufemismo: no modo
> padrão (`truncate`) ele **apaga a tabela inteira sem WHERE, sem confirmação e sem retorno de
> quantas linhas foram**. Não configurar sem antes ler `checklist-pre-producao`.

## O que faz

`[CONFIRMADO-MCP]` Limpa uma tabela SQL de destino via **TRUNCATE** ou **DELETE (com WHERE
opcional)** e **repassa a referência DuckDB upstream intacta**. Tipicamente usada **antes de um
`bulk_insert`** para recarregar a tabela. — `describe_node('truncate_table')`

`[VÍDEO]` Em aula: *"Limpar Tabela é como o nome já diz, né? Pra limpar tabela."* — `m3p2b:156`

## Quando usar

`[CONFIRMADO-MCP]`
- Limpar uma tabela de destino **antes de recarregar** dados (TRUNCATE + `bulk_insert`).
- **Deletar registros antigos com um WHERE filtrado** antes de inserir novos.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('truncate_table')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `connection_id` | string | **sim** | — | UUID do conector SQL de destino |
| `target_table` | string | **sim** | — | Tabela a limpar (ex.: `schema.tabela`) |
| `mode` | enum | não | **`truncate`** | `truncate` / `delete` |
| `where_clause` | string | não | — | Cláusula WHERE opcional **no modo `delete`** — **sem a palavra WHERE** |
| `output_field` | string | não | `data` | Nome do campo de saída |

### Os dois modos

`[UI-OBSERVADA]` Em aula os dois aparecem como escolha no painel: *"Então você tem duas opções:
TRUNCATE, mais rápido porém não te dá nenhum tipo de verificação, né? E o DELETE ele pode, você
pode colocar um WHERE"* — `m3p2b:159`

| Modo | Comportamento | Filtro |
|---|---|---|
| `truncate` | **Padrão.** Mais rápido; *"não te dá nenhum tipo de verificação"* `[VÍDEO]` | **Nenhum possível** |
| `delete` | `DELETE` com filtro | `where_clause` |

> ⚠️ **O padrão é o modo destrutivo.** `mode` é opcional e cai em `truncate`. Um nó adicionado ao
> canvas e salvo sem tocar em `mode` **apaga a tabela inteira**.

## ⚠️ Armadilha confirmada nas duas fontes: `where_clause` **não leva a palavra `WHERE`**

Esta é a rara armadilha em que aula e MCP **concordam exatamente**, e vale citar as duas porque a
aula mostra o sintoma e o MCP dá a regra.

`[VÍDEO]` O instrutor digitou a palavra `WHERE` dentro do campo e recebeu erro de sintaxe:

> *"vou botar lá, WHERE DEPARTAMENTO igual a VENDAS. Executar."* → *"(AÇÃO: Identificação de um erro
> de sintaxe nos resultados da execução decorrente do uso explícito da palavra WHERE)"* →
> *"**Opa, não precisa do WHERE aqui.**"* — `m3p2b:159-164`

`[CONFIRMADO-MCP]` A regra, literal no contrato: *"Cláusula WHERE opcional no modo 'delete'
(**sem a palavra WHERE**)"*.

`[UI-OBSERVADA]` A expressão que funcionou foi apenas o predicado: `DEPARTAMENTO = 'VENDAS'`
(reconstruído de `m3p2b:159-165` — a transcrição descreve *"removendo a palavra WHERE e deixando
apenas a expressão avaliada"*, sem mostrar o texto exato). `[LACUNA]` A sintaxe aceita no campo
(aspas simples? nome de coluna case-sensitive no Oracle?) não está literal em nenhuma fonte.

## Entradas esperadas

`[CONFIRMADO-MCP]` Qualquer upstream — a referência DuckDB é **repassada intacta**.

`[VÍDEO]` Confirmado na aula: *"Aqui ele só passa pra frente, né? Porque como é um nó que não faz
nada, não retorna dados, ele só passa o anterior pra frente."* — `m3p2b:166`

`[INFERIDO]` Isto é o que faz o nó **encaixar em série** antes de um `bulk_insert` sem quebrar o
dataset: `sql_database → truncate_table → bulk_insert` funciona porque o `truncate_table` não
consome nem altera o dataset.

## Saídas produzidas

`[CONFIRMADO-MCP]` O **mesmo dataset upstream**, no campo `output_field` (padrão `data`).

`[LACUNA]` **Não há relatório de quantas linhas foram apagadas.** Nem o contrato nem a aula
mencionam contagem de saída — a verificação em aula foi feita **fora da plataforma**, num cliente
de banco (`m3p2b:166-168`). Para o piloto isso significa que **o efeito do nó não é auditável de
dentro do fluxo**. Ver L43.

## Erros comuns

`[VÍDEO]` Palavra `WHERE` dentro do `where_clause` → erro de sintaxe. — `m3p2b:162-164`

`[INFERIDO]` `where_clause` preenchido com `mode: truncate` → **ignorado silenciosamente**. O
contrato diz que o campo vale "no modo `delete`", mas não declara o que acontece se for preenchido
no outro modo. **Não confirmado, e é um caminho de perda de dados**: quem preencher o filtro e
esquecer de trocar o modo apaga a tabela toda achando que filtrou. Ver L44, **impacto ALTO**.

`[CONFIRMADO-DOC]` **Não é exportável** para SQL/Python: `truncate_table` está entre os tipos de
"Efeitos colaterais" que devolvem **HTTP 422**.
— `guias-de-uso/exportar-e-importar.md § Cobertura V1`

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Limpar tabela de staging antes de recarregar
{"connection_id": "<uuid>", "target_table": "dbo.clientes_staging", "mode": "truncate"}
```

`[VÍDEO]` Montado em aula, contra a tabela `FUNCIONARIOS`:

```
mode: delete
where_clause: DEPARTAMENTO = 'VENDAS'
```

Resultado narrado: *"não temos mais nenhum nó que é do tipo Vendas"*, conferido no cliente de banco
(`m3p2b:166`). Em seguida o mesmo nó foi trocado para `truncate` e *"a tabela foi truncada, não tem
mais dado"* (`m3p2b:168-169`).
`[LACUNA]` A contagem citada na transcrição é ambígua (*"Antes 255, não temos mais, 300
registros"*, `m3p2b:166`) — não se consegue reconstruir quantas linhas foram apagadas.

## Observações para o piloto de margem

1. **Este nó não pertence ao caminho principal do piloto.** Corrigir preço é `UPDATE`, não
   recarga — e o alvo é uma tabela **do ERP**, onde `truncate` é impensável.
2. **Onde ele pode servir:** limpar uma **tabela de staging própria** do piloto entre execuções —
   a área onde a trilha de auditoria ou os candidatos fora-de-faixa da execução corrente ficam
   antes de serem consolidados. Padrão `truncate_table → bulk_insert` que o próprio contrato
   recomenda. `[INFERIDO]`
3. **`mode: delete` + `where_clause` é a peça para reprocessar uma janela.** Apagar
   `DATA_EMISSAO >= :inicio_janela` na tabela de auditoria antes de regravar dá idempotência à
   janela de sobreposição do `reprocess_window` (L6). `[INFERIDO]` — mas `where_clause` é string
   fixa no contrato, **sem `parameters`**: não há como ligar a janela a uma variável do fluxo.
   `[LACUNA]` O `where_clause` aceita template de variável (`{{vars.X}}`)? O contrato não diz.
   Ver L45, **impacto ALTO para a idempotência**.
   > `[CONFIRMADO-MCP]` Alternativa que **não** tem esse problema: `dataset_write` com
   > `mode: overwrite_partition` — *"substitui só as partições presentes no lote — é como se
   > reprocessa uma janela sem duplicar nem deixar buraco"*. Ver `nos/` e o relatório de L28.
4. **O padrão `truncate` é um risco de configuração, não de execução.** Toda skill que gere este nó
   deve **exigir `mode` explícito**, jamais deixar o default valer.
5. **Efeito não auditável de dentro do fluxo (L43)** — combina mal com a exigência do plano §5 de
   registrar antes/depois de cada alteração.
