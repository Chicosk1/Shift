---
title: Consolidação de Nós de Transformação
---

Este documento descreve a consolidação de 13 nós legados em 3 nós unificados:
**`transform`**, **`aggregate`** e **`combine`**.

Os nós legados continuam funcionando — não foram removidos. A migração é
opcional e pode ser feita gradualmente.

---

## Mapa de migração (antigo → novo)

| Nó legado | Nó consolidado | Step / modo |
| --- | --- | --- |
| `mapper` | `transform` | step `select_fields` |
| `math` | `transform` | step `compute` |
| `filter` | `transform` | step `filter` |
| `sort` | `transform` | step `sort` |
| `deduplication` | `transform` | step `deduplicate` |
| `record_id` | `transform` | step `add_id` |
| `sample` | `transform` | step `sample` |
| `unpivot` | `transform` | step `unpivot` |
| `text_to_rows` | `transform` | step `expand_text` |
| `aggregator` | `aggregate` | `config.mode = "group_by"` |
| `pivot` | `aggregate` | `config.mode = "pivot"` |
| `union` | `combine` | `config.mode = "union"` |
| `join` | `combine` | `config.mode = "join"` |

---

## Nó `transform` — 9 steps encadeados

O nó `transform` aplica uma lista ordenada de steps sobre uma única entrada.
Cada step é executado em sequência sobre o resultado do anterior, o que
elimina nós intermediários no canvas.

### Estrutura base

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [
    { "kind": "<step_kind>", ...campos_do_step }
  ]
}
```

---

### `select_fields` ← `mapper`

Renomeia, reordena ou descarta colunas. Suporta expressões computadas e cast
de tipo.

**Legado (`mapper`)**

```
{
  "type": "mapper",
  "mappings": [
    { "source": "nome_cliente", "target": "nome", "type": "string" },
    { "source": "val",          "target": "valor", "expression": "val * 1.1", "type": "float" }
  ],
  "drop_unmapped": true,
  "output_field": "data"
}
```

**Novo (`transform` / `select_fields`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "select_fields",
    "mappings": [
      { "source": "nome_cliente", "target": "nome", "type": "string" },
      { "source": "val",          "target": "valor", "expression": "val * 1.1", "type": "float" }
    ],
    "drop_unmapped": true
  }]
}
```

---

### `compute` ← `math`

Adiciona colunas calculadas via expressões SQL sem remover colunas existentes.

**Legado (`math`)**

```
{
  "type": "math",
  "expressions": [
    { "target_column": "margem", "expression": "(preco - custo) / preco" },
    { "target_column": "desconto_pct", "expression": "desconto / preco * 100" }
  ],
  "output_field": "data"
}
```

**Novo (`transform` / `compute`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "compute",
    "expressions": [
      { "target_column": "margem", "expression": "(preco - custo) / preco" },
      { "target_column": "desconto_pct", "expression": "desconto / preco * 100" }
    ]
  }]
}
```

---

### `filter` ← `filter`

Filtra linhas por condições. Suporta 16 operadores e lógica AND/OR.

**Legado (`filter`)**

```
{
  "type": "filter",
  "logic": "and",
  "conditions": [
    { "field": "status", "operator": "eq",  "value": "ativo" },
    { "field": "valor",  "operator": "gte", "value": 100 }
  ],
  "output_field": "data"
}
```

**Novo (`transform` / `filter`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "filter",
    "logic": "and",
    "conditions": [
      { "field": "status", "operator": "eq",  "value": "ativo" },
      { "field": "valor",  "operator": "gte", "value": 100 }
    ]
  }]
}
```

---

### `sort` ← `sort`

Ordena linhas por uma ou mais colunas, com controle de NULLs e limit opcional.

**Legado (`sort`)**

```
{
  "type": "sort",
  "sort_columns": [
    { "column": "data_venda", "direction": "desc", "nulls_position": "last" },
    { "column": "valor",      "direction": "asc" }
  ],
  "limit": 1000,
  "output_field": "data"
}
```

**Novo (`transform` / `sort`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "sort",
    "sort_columns": [
      { "column": "data_venda", "direction": "desc", "nulls_position": "last" },
      { "column": "valor",      "direction": "asc" }
    ],
    "limit": 1000
  }]
}
```

---

### `deduplicate` ← `deduplication`

Remove duplicatas via `ROW_NUMBER() OVER (PARTITION BY ...)`.

**Legado (`deduplication`)**

```
{
  "type": "deduplication",
  "partition_by": ["cliente_id", "produto_id"],
  "order_by": [{ "column": "data_venda", "direction": "desc" }],
  "keep": "first",
  "output_field": "data"
}
```

**Novo (`transform` / `deduplicate`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "deduplicate",
    "partition_by": ["cliente_id", "produto_id"],
    "order_by": [{ "column": "data_venda", "direction": "desc" }],
    "keep": "first"
  }]
}
```

---

### `add_id` ← `record_id`

Adiciona coluna de ID sequencial via `ROW_NUMBER()`.

**Legado (`record_id`)**

```
{
  "type": "record_id",
  "id_column": "id",
  "start_at": 1,
  "start_at_offset": 0,
  "partition_by": [],
  "order_by": [{ "column": "data_venda", "direction": "asc" }],
  "output_field": "data"
}
```

**Novo (`transform` / `add_id`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "add_id",
    "id_column": "id",
    "start_at": 1,
    "start_at_offset": 0,
    "partition_by": [],
    "order_by": [{ "column": "data_venda", "direction": "asc" }]
  }]
}
```

---

### `sample` ← `sample`

Amostragem: primeiras N linhas, N linhas aleatórias ou percentual.

**Legado (`sample`)**

```
{
  "type": "sample",
  "mode": "first_n",
  "n": 500,
  "output_field": "data"
}
```

**Novo (`transform` / `sample`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "sample",
    "mode": "first_n",
    "n": 500
  }]
}
```

Modos disponíveis: `first_n`, `random` (requer `seed`), `percent` (requer `percent`).

---

### `unpivot` ← `unpivot`

Transforma colunas em linhas (wide → long).

**Legado (`unpivot`)**

```
{
  "type": "unpivot",
  "index_columns": ["produto"],
  "value_columns": ["jan", "fev", "mar"],
  "variable_column_name": "mes",
  "value_column_name": "vendas",
  "cast_value_to": "float",
  "output_field": "data"
}
```

**Novo (`transform` / `unpivot`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "unpivot",
    "index_columns": ["produto"],
    "value_columns": ["jan", "fev", "mar"],
    "variable_column_name": "mes",
    "value_column_name": "vendas",
    "cast_value_to": "float"
  }]
}
```

Alternativa ao invés de listar colunas: `"by_type": "all_numeric"` ou `"by_type": "all_string"`.

---

### `expand_text` ← `text_to_rows`

Divide uma coluna de texto por delimitador, gerando uma linha por fragmento.

**Legado (`text_to_rows`)**

```
{
  "type": "text_to_rows",
  "column_to_split": "tags",
  "delimiter": ",",
  "output_column": "tag",
  "keep_empty": false,
  "trim_values": true,
  "output_field": "data"
}
```

**Novo (`transform` / `expand_text`)**

```
{
  "type": "transform",
  "output_field": "data",
  "steps": [{
    "kind": "expand_text",
    "column_to_split": "tags",
    "delimiter": ",",
    "output_column": "tag",
    "keep_empty": false,
    "trim_values": true
  }]
}
```

---

### Vantagem dos steps encadeados

Um único nó `transform` pode substituir múltiplos nós legados encadeados:

```
{
  "type": "transform",
  "output_field": "clientes_limpos",
  "steps": [
    { "kind": "filter",      "logic": "and", "conditions": [{ "field": "ativo", "operator": "eq", "value": true }] },
    { "kind": "compute",     "expressions": [{ "target_column": "nome_upper", "expression": "UPPER(nome)" }] },
    { "kind": "deduplicate", "partition_by": ["cpf"], "keep": "first" },
    { "kind": "sort",        "sort_columns": [{ "column": "nome_upper", "direction": "asc" }] }
  ]
}
```

---

## Nó `aggregate` — group_by e pivot

### `group_by` ← `aggregator`

**Legado (`aggregator`)**

```
{
  "type": "aggregator",
  "group_by": ["regiao", "categoria"],
  "aggregations": [
    { "operation": "sum",   "column": "valor",    "alias": "total_valor" },
    { "operation": "count", "column": "*",         "alias": "qtd" },
    { "operation": "avg",   "column": "desconto",  "alias": "media_desconto" }
  ],
  "output_field": "data"
}
```

**Novo (`aggregate` / `group_by`)**

```
{
  "type": "aggregate",
  "output_field": "data",
  "config": {
    "mode": "group_by",
    "group_by": ["regiao", "categoria"],
    "aggregations": [
      { "operation": "sum",   "column": "valor",    "alias": "total_valor" },
      { "operation": "count", "column": "*",         "alias": "qtd" },
      { "operation": "avg",   "column": "desconto",  "alias": "media_desconto" }
    ]
  }
}
```

---

### `pivot` ← `pivot`

**Legado (`pivot`)**

```
{
  "type": "pivot",
  "index_columns": ["produto"],
  "pivot_column": "mes",
  "value_column": "vendas",
  "aggregations": ["sum"],
  "max_pivot_values": 200,
  "output_field": "data"
}
```

**Novo (`aggregate` / `pivot`)**

```
{
  "type": "aggregate",
  "output_field": "data",
  "config": {
    "mode": "pivot",
    "index_columns": ["produto"],
    "pivot_column": "mes",
    "value_column": "vendas",
    "aggregations": ["sum"],
    "max_pivot_values": 200
  }
}
```

---

## Nó `combine` — union e join

### `union` ← `union`

> **Atenção à renomeação**: o campo `mode` do nó legado (`"by_name"` /
> `"by_position"`) vira `align` no novo nó. O campo `mode` do novo nó é
> sempre `"union"` (discriminador de operação).

**Legado (`union`)**

```
{
  "type": "union",
  "mode": "by_name",
  "add_source_col": true,
  "source_col_name": "origem",
  "dedup_keys": ["id"],
  "dedup_priority": [{ "source": "input_1", "priority": 1 }],
  "output_field": "data"
}
```

**Novo (`combine` / `union`)**

```
{
  "type": "combine",
  "output_field": "data",
  "config": {
    "mode": "union",
    "align": "by_name",
    "add_source_col": true,
    "source_col_name": "origem",
    "dedup_keys": ["id"],
    "dedup_priority": [{ "source": "input_1", "priority": 1 }]
  }
}
```

Handles de entrada: dinâmicos (`input_1`, `input_2`, ..., `input_N`).

---

### `join` ← `join`

**Legado (`join`)**

```
{
  "type": "join",
  "join_type": "left",
  "conditions": [
    { "left_column": "cliente_id", "right_column": "id" }
  ],
  "columns": null,
  "output_field": "data"
}
```

**Novo (`combine` / `join`)**

```
{
  "type": "combine",
  "output_field": "data",
  "config": {
    "mode": "join",
    "join_type": "left",
    "conditions": [
      { "left_column": "cliente_id", "right_column": "id" }
    ],
    "columns": null
  }
}
```

Handles de entrada: fixos `left` e `right`.

---

## Quando usar o nó consolidado vs. o legado

| Situação | Recomendação |
| --- | --- |
| Novo workflow sendo criado | Use os nós consolidados |
| Workflow legado funcionando em produção | Não migre — legado continua suportado |
| Precisa de 3+ transformações seguidas sobre a mesma tabela | `transform` com múltiplos steps |
| Apenas uma transformação simples e o workflow já usa legado | Mantenha o legado |
| Alternância dinâmica entre group_by e pivot na mesma UI | `aggregate` |
| Empilhamento de N datasets (>2 fontes) | `combine` modo `union` |
| Cruzamento de dois datasets com chave | `combine` modo `join` |

---

## Utilitário de migração programática

Para migrar workflows via código, use o utilitário em
`app/services/workflow/migration/migrate_to_consolidated.py`:

```
from app.services.workflow.migration.migrate_to_consolidated import migrate_workflow

# workflow_definition é o dict salvo no campo `definition` do modelo Workflow
new_definition = migrate_workflow(workflow_definition)
```

A função é **não-destrutiva**: retorna uma cópia nova, não modifica o original.
Nós que não são legados (ex: `sql_database`, `bulk_insert`) são passados
inalterados.
