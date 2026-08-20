# Divergências — aulas e documentação vs. MCP

**Fase 2, 2026-08-20.** Regra: onde houver conflito, **vence o MCP**.

> **Alcance desta comparação.** `conhecimento/extraido/` tem só o **lote 1** (glossário, 4 nós, 3
> procedimentos). Onde o material de aula já foi inventariado mas ainda não extraído, comparo
> contra o inventário da Fase 0 e marco a fonte como *(inventário, não extraído)* — para não
> passar por confirmado o que ainda não foi lido em profundidade.

**Escopo de banco: somente Oracle.** Divergências que só afetam SQL Server, Firebird, Postgres ou
MySQL estão marcadas *(fora de escopo)*.

---

## 0. Leituras executadas no MCP

Todas read-only. **Nenhuma operação de criação, alteração ou remoção foi executada.**

| Ferramenta | Resultado |
|---|---|
| `list_projects` | **Nenhum projeto** |
| `list_workflows` | **Nenhum workflow** |
| `list_webhooks` | **Nenhum webhook** |
| `listar_bases_internas` | **Nenhuma base interna** |
| `list_connections` | **2 conexões Oracle** |
| `get_connection` ×2 | Metadados abaixo |
| `test_connection` ×2 | **SUCESSO** nas duas |
| `list_nodes` | 63 tipos; buscas por "conjunto" e "parquet dataset gravar" |
| `describe_node` ×10 | `manual`, `csv_input`, `mapper`, `bulk_insert`, `cron`, `sql_database`, `sql_script`, `if_node`, `math`, `internal_data_write` |

### O ambiente real alcançado pela chave — **fecha L20**

Workspace `fd9cff30-2988-48db-a19b-9faba7ea06c4`, contendo **apenas conexões**:

| Nome | Tipo | Host | Porta | Banco | Usuário | Pública |
|---|---|---|---|---|---|---|
| Viasuper Padrão - Gabriel | `oracle` | 192.168.90.218 | 30200 | ORCL | VIASOFTMERC | Sim |
| Viasuper Titan - Gabriel | `oracle` | 192.168.90.218 | 30100 | ORCL | VIASOFTMERC | Sim |

Não é o workspace `Treinamento` das aulas: **zero projetos, fluxos, webhooks e bases internas.**
É um ambiente limpo com duas conexões Oracle prontas — coerente com o escopo Oracle do projeto.

### `test_connection` responde a L14 — relay não é necessário

**As duas conexões retornaram SUCESSO.** Nenhuma delas indica relay nos metadados, e a conexão
real ao Oracle em `192.168.90.218` funciona. Para **estas** conexões, o Oracle é alcançável
direto — **relay não é necessário**.
`[LACUNA remanescente]` As duas apontam para o mesmo host, provavelmente ambiente de
desenvolvimento. Falta confirmar se o Oracle de **produção** do ERP é igualmente alcançável.

---

## 1. Confirmado — existe nas aulas/doc E no MCP, coerente

| Item | Fonte original | Confirmação no MCP |
|---|---|---|
| Conexão nunca devolve senha | `conceitos.md § Conexão` | `get_connection` devolve host/porta/banco/usuário e declara *"Nunca retorna senhas, tokens ou strings de conexão"* — verificado nas 2 conexões |
| Oracle é suportado | `conceitos.md § Conexão`, `m1` | `list_connections` traz `tipo: oracle`; `test_connection` SUCESSO |
| Estratégias do Bulk Insert | `primeiros-passos.md §4` | `bulk_insert`: `append_fast`, `append_safe`, `upsert`, `insert_if_not_exists` — as 4, exatas |
| Upsert/insert_if_not_exists exigem chave | `primeiros-passos.md §4` | `merge_keys` no contrato |
| Modos da Base Interna | `primeiros-passos.md §4` | `internal_data_write`: `insert`, `upsert`, `update`, `delete`, `replace` — os 5, exatos |
| Base interna: teto de 200 mil linhas | `primeiros-passos.md §4` | Confirmado e **nomeado**: `INTERNAL_DATA_MAX_ROWS`, hoje 200.000 |
| Modos do SQL Script | `m3-2B` *(inventário)* | `sql_script`: `query`, `execute`, `execute_many` — os 3, exatos |
| `sql_database` faz streaming e leitura particionada | `m3-2B` *(inventário)*, `benchmark-streaming-partition.md` | `chunk_size`, `partition_on`, `partition_num` |
| Gatilho não tem entrada | `m1:73`, `conceitos.md § Nó` | `manual` e `cron` não têm parâmetro de entrada tabular |
| Cron devolve o timestamp do disparo | `nos/agendamento-(cron).md` | `cron`: *"Retorna timestamp do disparo como output"* |
| Mapper: transforms de texto | `m3-3A` *(inventário)* | `upper`, `lower`, `trim`, `strip_accents`, `replace`, `regexp_replace` etc. |
| `math` cria coluna nova; `mapper` sobrescreve | `m3-3E` *(inventário)* | `math` tem `target_column` por expressão; `mapper` tem `source`+`target` |
| CSV: delimitador, cabeçalho, encoding | `primeiros-passos.md §2`, `m1:66` | `delimiter`, `has_header`, `encoding` |
| `.xls` não suportado | `nos/excel.md` | Repetido no contrato do `excel_input` |
| Subfluxo roda isolado | `nos/entrada-do-fluxo.md` | `call_workflow`: *"contexto próprio, sem herdar upstream do pai"* |
| `if_node` tem gate mode | — (só doc de nó) | Confirmado, e detalhado com `decision_mode` |
| Papéis legados vivos | `m2p2` *(inventário)* | `create_project` exige `MANAGER`; `create_workflow` exige `CONSULTANT`+`EDITOR`; `list_project_members` devolve `EDITOR`/`CLIENT` |

---

## 2. Divergente — existe em ambos, com nome, parâmetro ou comportamento diferente

### D1 — Hierarquia: 4 níveis ou 3? *(aberta desde a Fase 0)*
- `Introducao/conceitos.md § A hierarquia` + `m1:41-55`: **4 níveis**, Organização → Espaço →
  Grupo econômico → Projeto.
- `controle-de-acesso.md § Hierarquia`: **3 níveis** de acesso, e *"Cliente continua existindo
  como entidade, mas não tem mais membership de acesso"*.
- `entidade-cliente.md`: terceira variante, de dados.

**MCP:** não desempata. Há `list_projects`, `get_project`, `list_project_members` e um
`workspace_id` nos metadados de conexão — mas **nenhuma ferramenta de grupo econômico/cliente**.
Isso é evidência *fraca* a favor da leitura de `controle-de-acesso.md` (cliente não é nível de
acesso), não prova.
**Segue aberta — L9.**

### D2 — O MCP expõe 48 ferramentas, não 15
`agente-shift-funcionamento.md` §4 (2026-04-20) lista 15. **São 48.** A doc está ~4 meses
defasada. Categorias ausentes da doc: build session `pending_*` (10), edição direta de fluxo (8),
documentação de fluxo (2), diagnóstico de rede local (2), protocolo de interação (2), bases
internas (2), avaliação de fluxo (1).

### D3 — Consolidação de nós de transformação não aconteceu como previsto
`consolidacao-de-nos-de-transformacao.md` prevê 13 nós legados fundidos em
`transform`/`aggregate`/`combine`. **`mapper`, `filter`, `sort`, `sample`, `record_id`, `union`,
`pivot`, `unpivot`, `text_to_rows` seguem separados.** Não existe `transform` nem `aggregate`
(existe `aggregator`, que é o legado). Só `combine` foi criado, e a descrição de `join` diz
*"Versão legada — prefira o nó `combine` com `mode='join'`"*.
**Valida a decisão** de usar o nome legado como chave primária. **L12 fechada.**

### D4 — Papéis: doc diz que renomeou; MCP usa os nomes antigos
`controle-de-acesso.md` afirma `MANAGER`→`ADMIN`, `CONSULTANT`→`EDITOR`, `OPERATOR`→`VIEWER`, com
422 para role inválido. O MCP fala `MANAGER`, `CONSULTANT`, `CLIENT`. Coerente com a ressalva da
própria doc: a migração `c7d8e9f0a1b2` é **destrutiva e não aplicada**.
**Ação:** não codificar matriz de permissão na skill. **L18.**

### D5 — `Gravar Base` foi removido
`m3-2B` *(inventário)*: o narrador cita `Gravar Base` e diz *"acho que até ele tá
descontinuado"*. **Não existe no MCP.** A aula estava certa.

### D6 — As duas ferramentas de variável não aceitam os mesmos tipos
`pending_set_variables` aceita 10 tipos (`string`, `number`, `integer`, `boolean`, `object`,
`array`, `table_reference`, `connection`, `file_upload`, `secret`). `set_workflow_variables`
(escrita direta) aceita **6** — ficam de fora `table_reference`, `connection`, `file_upload`,
`secret`.
**Consequência:** declarar variável de **conexão ou de segredo** só é possível pela build session.
E é a ferramenta direta que parece mais simples. **L11.**

### D7 — Cron: a UI tem piso de 5 minutos, o MCP aceita expressão livre ⚠️
- `nos/agendamento-(cron).md` `[CONFIRMADO-DOC]`: a expressão é **gerada por interface visual**
  a partir de frequências prontas, sendo *"A cada 5 minutos"* → `*/5 * * * *` a menor. *"Não é
  necessário saber escrever cron manualmente."* Não há opção de 1 minuto.
- **MCP:** `cron` tem **um único parâmetro** — `cron_expression` (string, obrigatório), com o
  exemplo `'0 8 * * 1-5'`. **Nenhum piso, nenhuma lista de valores permitidos.**

**Leitura:** o piso de 5 minutos é da UI, não necessariamente do motor (APScheduler aceita
`* * * * *`). **Resolve L15 parcialmente** — via MCP dá para escrever cron arbitrário.
`[LACUNA]` Falta saber se o backend valida o intervalo mínimo ao salvar, ou se aceita e o
scheduler simplesmente dispara mais rápido. **Não testável sem escrita.**

### D8 — Marca d'água é NATIVA do `sql_database` ⚠️ **o achado mais importante**
`plano-skills-shift.md §5.1` manda persistir o timestamp do último registro processado à mão, e
somar uma margem de sobreposição. `lacunas.md` L3 registrava watermark como lacuna de impacto
alto, com **zero ocorrências** em 380 KB de transcrição e 40 documentos.

**O MCP mostra que o nó já faz isso:**

| Parâmetro | O que faz |
|---|---|
| `incremental_column` | Coluna de corte (data, timestamp ou sequencial crescente) |
| `initial_value` | De onde começar na **primeira** execução. Vazio = lê tudo |
| `reprocess_window` | **Recuo aplicado ao marco a cada execução** — dias se a coluna for data/hora, unidades se sequencial. *"Existe para pegar lançamento retroativo"* |

E os nomes `incremental_from`, `shift_p_lo`, `shift_p_hi` são **reservados pelo motor** (leitura
incremental / particionamento) e recusados como nome de parâmetro — o marco é gerenciado
internamente.

`reprocess_window` **é exatamente** a "margem de sobreposição" que §5.1 pede, como parâmetro de
configuração. **L3 essencialmente fechada.**

> ⚠️ Armadilha declarada: `incremental_column` **precisa de índice na origem** — *"sem ele o
> incremental vira varredura completa e o ganho some"*.

### D9 — Parametrização: bind real num nó, substituição literal no outro ⚠️
Os dois parecem a mesma coisa e têm segurança oposta:

| Nó | Mecanismo | Comportamento |
|---|---|---|
| `sql_database`, `sql_script` | `:nome` + `parameters` | *"bind de verdade, sem interpolação de texto"* |
| `mapper` | `{{vars.X}}` em `expression` | *"substituído pelo VALOR LITERAL antes do SQL rodar"* — texto **exige** aspas simples ou quebra a sintaxe |

Nenhuma aula nem documento registra a diferença. É uma pegadinha de segurança e de correção.

### D10 — Escopo de conexão: doc tem 3 valores, MCP tem booleano + workspace
`conceitos.md § Conexão` descreve escopo de **3 valores** (Espaço / Grupo econômico / Projeto),
com *"toda conexão pertence a exatamente um dono"*. O `get_connection` devolve **`Publica: Sim`**
e **`Workspace: <uuid>`** — não um campo de escopo de 3 valores, e nenhuma referência a grupo
econômico ou projeto.
`[LACUNA]` Falta saber se `Publica` + ausência de `client_id`/`project_id` **é** a representação
do escopo "Espaço", ou se o MCP simplesmente não expõe os outros dois.

### D11 — `sql_script` tem um terceiro nível de risco
Os níveis observados são `read_only` (manual, csv_input, mapper, math, if_node, cron,
sql_database) e `write` (bulk_insert, internal_data_write). O `sql_script` é
**`unknown_write`** — a plataforma **não sabe estaticamente** se o script escreve.
Não aparece em nenhuma doc. Relevante para qualquer checklist automatizado de pré-produção.

---

## 3. Só em um lado

### 3.1 Só no MCP — e importa para o piloto

| Recurso | Nó / ferramenta | Por que importa |
|---|---|---|
| `reprocess_window`, `incremental_column`, `initial_value` | `sql_database` | Marca d'água nativa — ver D8 |
| `on_update`: `overwrite` / `keep_if_empty` / `never` | `bulk_insert` | Idempotência por coluna, sem lógica no fluxo (L6) |
| `unique_columns` | `bulk_insert` | Deduplica **antes** do insert — resolve a duplicação criada pela sobreposição da janela |
| `decision_mode: single` | `if_node` | *"avalia SÓ a 1ª linha e envia a tabela inteira por um único ramo"* — **é o dry-run e o disjuntor** (L2, L5) |
| `returning_columns` | `bulk_insert` | Captura o valor gravado para a trilha de auditoria |
| `execute_many` | `sql_script` | Itera sobre o upstream. Exemplo do contrato é literalmente `UPDATE ... WHERE id = :id` |
| `read_delivery_mode` / `delivery_mode` | `sql_database`, `bulk_insert` | `auto`/`tunnel`/`edge` — controle do caminho de borda |
| `timeout_seconds` (padrão 60) | `sql_script` | Timeout por nó, muito abaixo do 3600 s do fluxo |
| `avaliar_fluxo` | ferramenta | Detecta **SQL destrutivo sem proteção** e caminho de erro solto |
| `ler_rejeicoes` | ferramenta | Dead-letter por causa, com campos sensíveis mascarados |
| `perguntar`, `planejar` | ferramentas | A plataforma tem protocolo próprio de pausar e planejar |

### 3.2 Só no MCP — armadilha Oracle que nenhuma fonte registra ⚠️

Do contrato do `sql_script`, e **crítica no escopo Oracle**:

> Nunca use `''` para dizer 'sem valor': **no Oracle string vazia JÁ É NULL**, então o filtro muda
> de sentido em silêncio conforme o banco. Se o valor é opcional, tire o trecho do SQL em vez de
> mandar vazio.

E, em `sql_database` e `sql_script`:
- Placeholder **dentro de comentário** conta como parâmetro obrigatório — `-- filtra por :cnpj`
  vira bind exigido.
- Parâmetro que resolve para `None` é **recusado antes de tocar o banco**.

### 3.3 Só nas aulas/doc — não aparece no MCP

| Item | Onde é citado | Status |
|---|---|---|
| **Gatilho "Monitorar Mudanças" / "Monitoramento" / "Monitorar tabelas"** | `conceitos.md § Nó`, `m1:65`, `m3-2A` | `list_nodes` só devolve `manual`, `webhook`, `cron`. **L22, impacto alto** |
| **Dead Letter como tipo de nó** | `conceitos.md § Nó`, `exportar-e-importar.md` | Não é nó. É recurso de execução, via `ler_rejeicoes`. **L10** |
| Gmail (enviar e-mail), WhatsApp via Z-API | `conceitos.md § Nó`, `visao-geral.md` | Ausentes. Afeta `sub-notificar-falha`. **L23** |
| Analista IA, Decisão IA | `conceitos.md § Nó` | Ausentes. **L23** |
| Nota, Grupo | `conceitos.md § Nó` | Provavelmente elementos de canvas, não nós de engine |
| Nós `transform` e `aggregate` consolidados | `consolidacao-...md` | Não existem. Ver D3 |
| `Destino SQL` | `conceitos.md § Nó` | Provável rótulo de UI do `loadNode`. **A confirmar com `describe_node('loadNode')`** |
| "Integrações" como categoria | `conceitos.md § Nó` | A descrição de `list_nodes` diz explicitamente que **não é** categoria |
| Prefixo `shk_` de API Key | `agente-shift-funcionamento.md` | A chave em uso tem prefixo diferente |

### 3.4 Inconsistência interna do MCP ⚠️

O contrato do `internal_data_write` recomenda, para histórico e volume:

> Para histórico, resultado de processamento ou qualquer volume grande o destino certo é o nó
> **'Gravar em Conjunto de Dados'** (parquet, lido como tabela pelos fluxos).

**Esse nó não existe em `list_nodes`.** Verifiquei com busca por `conjunto` (devolveu `filter`,
`sample`, `stat_market_basket` — falsos positivos) e por `parquet dataset gravar` (nenhum
resultado).

**Por que importa muito:** é o destino recomendado para a **trilha de auditoria** do piloto, que
por natureza é histórico e cresce sem parar — e a base interna tem teto duro de 200 mil linhas.
Sem esse nó, não há destino óbvio para a auditoria. **Nova lacuna L28.**

---

## 4. Operações de escrita que eu precisaria — e não executei

Conforme combinado, apenas listo:

### 4.1 Descobrir o schema do ERP (tabelas de pedido, item, preço, margem)
**Não existe ferramenta de leitura de schema no MCP.** Não há `list_tables`, `describe_table` nem
`get_schema` entre as 48. As alternativas seriam:

- `create_workflow` + `add_node`(`sql_database`) + `execute_workflow` — **três operações de
  escrita** só para rodar um SELECT em catálogo.

**Recomendação: não autorize.** Use o **Playground** da conexão na interface (que roda só
`SELECT`) e me passe o resultado — chego ao mesmo lugar sem escrever nada. As consultas úteis
seriam sobre `ALL_TABLES` / `ALL_TAB_COLUMNS` do schema `VIASOFTMERC`, filtrando por nome de
tabela com "PEDIDO", "ITEM", "PRECO" e "MARGEM".

### 4.2 Confirmar se o cron aceita intervalo menor que 5 minutos (D7)
Exigiria `create_workflow` + `add_node`(`cron`) com `*/1 * * * *` para ver se o backend recusa.
**Recomendação: não autorize agora** — é curiosidade, o piloto pede 5 minutos, que é permitido.

### 4.3 Verificar a existência do nó de Conjunto de Dados (L28)
Exigiria tentar `add_node` com um tipo adivinhado. **Não recomendo adivinhar tipo de nó.**
Melhor caminho: olhar a biblioteca de nós na interface, na categoria de saída/banco.
