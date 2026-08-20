# Divergências — documentação e aulas vs. MCP

> Semeado na **Fase 0** (2026-08-20) com o que o dump do MCP já provou.
> A Fase 2 completa isto confrontando `conhecimento/extraido/` item por item.
>
> **Regra:** onde houver conflito, **vence o MCP**.

---

## 1. Confirmado (aula/doc coerente com o MCP)

| Item | Fonte original | Confirmação |
|---|---|---|
| Estratégias do Bulk Insert | `primeiros-passos.md` | `bulk_insert` expõe `append_fast`, `append_safe`, `upsert`, `insert_if_not_exists` — exatamente as 4 |
| Modos da Base Interna | `primeiros-passos.md` | `internal_data_write`: `insert`, `upsert`, `update`, `delete`, `replace` |
| Modos do SQL Script | `m3-2B` (aula) | `sql_script`: `query`, `execute`, `execute_many` |
| `.xls` não suportado | `nos/excel.md` | `excel_input` repete a restrição |
| Trigger cron devolve timestamp | `nos/agendamento-(cron).md` | `cron` confirma |
| Webhook expõe method/headers/query/body | `nos/webhook-(gatilho).md` | `webhook` confirma |
| Subfluxo roda isolado | `nos/entrada-do-fluxo.md` | `call_workflow`: "contexto próprio, sem herdar upstream do pai" |
| Sample com 3 modos | `nos/sample-(amostra).md`, `3G` | `first_n` / `random` (reservoir+seed) / `percent` (Bernoulli) |
| Union por nome/posição | `nos/union-(juntar).md`, `3I` | `by_name` / `by_position` |
| `truncate_table` repassa upstream | `m3-2B` | Confirmado |

---

## 2. Divergente

### D-MCP-1 — O MCP expõe 48 ferramentas, não 15
**Doc:** `agente-shift-funcionamento.md` §4 lista 15, em 5 grupos.
**MCP:** 48. Categorias inteiras ausentes da doc: build session `pending_*` (10), edição direta
de fluxo (8), documentação de fluxo (2), diagnóstico de rede local (2), protocolo de interação
(2), bases internas (2), avaliação de fluxo (1).
**Vence o MCP.** A doc é de 2026-04-20 e descreve um estágio anterior do produto.

### D-MCP-2 — O MCP **cria e edita** fluxo
**Registro anterior:** `lacunas.md` L13 supunha, a partir da doc, que não houvesse
`create_*`/`update_*` e que a construção fosse necessariamente manual.
**MCP:** `create_workflow`, `create_project`, `add_node`, `add_edge`, `remove_node`,
`remove_edge`, `update_node_config`, `set_workflow_variables`, `criar_base_interna`,
`escrever_documentacao`, mais a família `pending_*`.
**Impacto:** a expectativa da Fase 5 muda — o Claude Code pode construir, não apenas desenhar.

### D-MCP-3 — A consolidação de nós de transformação **não aconteceu como previsto**
**Doc:** `consolidacao-de-nos-de-transformacao.md` descreve 13 nós legados fundidos em
`transform` / `aggregate` / `combine`.
**MCP:** `mapper`, `filter`, `sort`, `sample`, `record_id`, `union`, `pivot`, `unpivot`,
`text_to_rows` **seguem existindo separados**. Não há nó `transform` nem `aggregate` (existe
`aggregator`, que é o legado). Só `combine` foi criado — e a descrição de `join` diz
explicitamente "Versão legada — prefira o nó `combine` com `mode='join'`".
**Conclusão:** a consolidação está parcial, restrita a union+join. **Valida a decisão** de usar
o nome legado como chave primária da base de conhecimento.

### D-MCP-4 — Vocabulário de papéis: o MCP usa os nomes legados
**Doc:** `controle-de-acesso.md` afirma que `MANAGER`→`ADMIN`, `CONSULTANT`→`EDITOR`,
`OPERATOR`→`VIEWER`, e que role inválido devolve 422.
**MCP:** `create_project` exige "role MANAGER"; `create_workflow` exige "CONSULTANT no
workspace e EDITOR no projeto"; `list_project_members` devolve "EDITOR ou CLIENT".
**Ou seja:** `MANAGER`, `CONSULTANT` e `CLIENT` estão vivos na superfície do MCP. Coerente com a
ressalva da própria doc de que a migração `c7d8e9f0a1b2` é **destrutiva e ainda não aplicada**.
**Ação:** não codificar matriz de permissão na skill até resolver. Alto risco de ficar errada.

### D-MCP-5 — `Gravar Base` (dito descontinuado na aula)
**Aula:** `m3-2B` menciona `Gravar Base` e o narrador diz "acho que até ele tá descontinuado".
**MCP:** não existe `Gravar Base`. Existe `internal_data_write` ("Base de Dados Interna
(Escrita)") e `loadNode` ("Carga SQL"). **A aula estava certa** — o nó saiu.

### D-MCP-6 — As duas ferramentas de variável não aceitam os mesmos tipos
**MCP, incoerência interna.** `pending_set_variables` (build session) aceita 10 tipos: `string`,
`number`, `integer`, `boolean`, `object`, `array`, `table_reference`, `connection`,
`file_upload`, `secret`. Já `set_workflow_variables` (escrita direta) aceita **só 6** — os 4
últimos ficam de fora.
**Consequência prática:** declarar variável de conexão, de segredo ou de arquivo **só é possível
pela build session**. A ferramenta direta silenciosamente não oferece esses tipos, e é a que
parece mais simples de usar. Codificar isso como regra na Fase 4.

---

## 3. Só no MCP (existe e não aparece em aula nem doc)

**23 dos 63 nós** não têm cobertura no material. Os que importam:

| Nó | Por que importa |
|---|---|
| **12 nós `stat_*`** | Categoria `statistics` inteira: Curva ABC, RFM, anomalias, previsão, ponto de pedido, coorte, correlação, cesta de compras, matriz de transição, PSI, WoE/IV, KS/Gini/AUC. Maior lacuna de cobertura do acervo |
| `composite_insert` | Cascata header+filhos em **uma transação** com FK via RETURNING. Feito para pedido+itens |
| `code` | Python 3.12 em Docker efêmero, sem rede, FS read-only |
| `switch_node`, `sync` | Partição por valor em N ramos; convergência de ramos paralelos |
| `loop` | Iteração com políticas de erro e limite — mecanismo de contenção |
| `text_search` | Busca por palavra, tolerante a acento e erro de digitação |
| `ofx_input`, `xml_input`, `xml_to_rows` | Extrato bancário e XML (inclui resposta de API não-JSON) |
| `inline_data` | Tabelas de domínio embutidas no fluxo |
| `pdf_report` | Relatório com KPIs e gráficos |
| `google_drive_download`, `google_drive_list` | Drive como fonte de arquivo, usável dentro de Loop |
| `combine` | Sucessor de join+union |

Ferramentas de MCP sem cobertura documental que mudam o modo de trabalhar:

- `avaliar_fluxo` — check-up determinístico que já detecta **SQL destrutivo sem proteção** e
  **caminho de erro solto**. A Fase 4 deve integrá-lo ao checklist de pré-produção em vez de
  reescrever a verificação à mão.
- `ler_rejeicoes` — dead-letter por causa, com campos sensíveis mascarados.
- `perguntar` / `planejar` — a plataforma tem opinião sobre pausar para perguntar e planejar
  antes de construir.
- `comando_para_maquina_local` / `ler_saida_colada` — diagnóstico de rede guiado, com foco em
  SQL Server.

---

## 4. Só na doc (não aparece no MCP)

| Item | Situação |
|---|---|
| Nós `transform` e `aggregate` consolidados | Não existem. Ver D-MCP-3 |
| `dead_letter` como **tipo de nó** | `conceitos.md` e `exportar-e-importar.md` citam; `list_nodes` não devolve. Dead-letter parece ser recurso de execução (via `ler_rejeicoes`), não nó |
| `Destino SQL` | Provável nome de interface do `loadNode`. **A confirmar com `describe_node`** |
| Prefixo `shk_` de API Key | A chave em uso tem prefixo diferente. Doc defasada ou tipo distinto de credencial |

---

## 5. A confirmar na Fase 2

1. `describe_node` em cada nó usado pelo piloto — o contrato completo de config.
2. Qual workspace a API Key alcança: `list_projects` devolveu vazio.
3. Se `Destino SQL` é o rótulo de `loadNode`.
4. Papéis reais em vigor (D-MCP-4) — pergunta para o suporte, não inferível.
5. Execução concorrente do mesmo fluxo agendado (L1) — segue sem resposta no MCP.
