# Catálogo de nós — dump real do MCP

> `[CONFIRMADO-MCP]` — `list_nodes` em 2026-08-20 (Fase 0). **63 tipos de nó.**
> Este é o índice compacto. O contrato completo de cada nó (campos de config, transforms,
> exemplos) vem de `describe_node`, a ser feito por nó na Fase 1.

Contagem por categoria: trigger 3 · input 11 · transform 18 · statistics 12 · logic 6 ·
database 6 · output 5 · subflow 3.

---

## trigger (3)

| Tipo | Rótulo | Observação |
|---|---|---|
| `manual` | Trigger Manual | Botão Play; repassa `input_data`; payload de fallback no nó |
| `webhook` | Trigger Webhook | Rota `/api/v1/webhook/{path}`; expõe method, headers, query_params, body |
| `cron` | Agendamento (Cron) | Devolve timestamp do disparo como output |

## input (11)

| Tipo | Rótulo | Observação |
|---|---|---|
| `sql_database` | Extração SQL | SELECT em banco externo. **Streaming e leitura particionada paralela** |
| `http_request` | Requisição HTTP | GET/POST/PUT/DELETE; auth Bearer/Basic/API Key; templates de variável |
| `csv_input` | Entrada CSV | Local ou remoto (HTTP/S3) |
| `excel_input` | Entrada Excel | `.xlsx` via streaming. **`.xls` não suportado** |
| `ofx_input` | Entrada OFX (Extrato) | Uma linha por transação (FITID, data, valor assinado). OFX 1.x e 2.x |
| `xml_input` | Entrada XML | Uma linha por ocorrência do elemento; leitura em fluxo |
| `inline_data` | Dados Embutidos | Tabelas de domínio, fixtures, valores de referência |
| `internal_data_source` | Base de Dados Interna (Leitura) | Sem conexão externa |
| `google_drive_download` | Google Drive (Baixar arquivo) | Devolve `shift-upload://<id>`; aceita valor do passo anterior (uso em Loop) |
| `google_drive_list` | Google Drive (Listar arquivos) | Uma linha por arquivo; filtro por extensão exata |
| `google_sheets_input` | Google Sheets (Ler) | Paginação automática |

## transform (18)

| Tipo | Rótulo | Observação |
|---|---|---|
| `mapper` | Mapper (Renomear/Selecionar) | `source` ou `expression` (SQL DuckDB em runtime) |
| `filter` | Filtrar Linhas | Operadores SQL com lógica AND/OR |
| `text_search` | Buscar em Texto | Por **palavra**, não substring; ignora acento/caixa, tolera erro de digitação, ordena por relevância |
| `aggregator` | Aggregator (Group By) | SUM/AVG/COUNT/MAX/MIN + `string_agg`. `group_by` vazio agrega tudo |
| `math` | Matemático | Colunas calculadas por expressão SQL DuckDB |
| `join` | Join (Cruzamento) | **LEGADO — a própria descrição manda preferir `combine` com `mode='join'`** |
| `combine` | Combinar (Union / Join) | Sucessor de `join` e `union`. Join aceita **cross** |
| `deduplication` | Deduplicar | `ROW_NUMBER() OVER (PARTITION BY)`; escolhe primeira/última |
| `sort` | Ordenação | Direção, posição de nulos, limite dos N primeiros |
| `sample` | Amostra | `first_n` / `random` (reservoir com seed) / `percent` (Bernoulli) |
| `record_id` | ID de Registro | `ROW_NUMBER() OVER()`; partição, ordenação, ponto de início |
| `union` | União | UNION ALL; `by_name` (NULL nas ausentes) ou `by_position` |
| `pivot` | Pivot | Descobre valores em runtime; CASE WHEN |
| `unpivot` | Unpivot (Wide → Long) | Explícito ou `all_numeric`/`all_string` |
| `text_to_rows` | Texto em Linhas | `UNNEST(string_split(...))`. **Substitui a coluna original** |
| `text_to_columns` | Texto em Colunas | Posicional; nº de colunas fixo ou descoberto, com teto de segurança |
| `unnest` | Desaninhar | Explode lista em linhas; structs viram colunas |
| `nest` | Aninhar | Constrói JSON hierárquico por níveis. Saída pronta para body de HTTP |
| `xml_to_rows` | XML em Linhas | Para coluna que **contém** XML — resposta de API não-JSON chega em `value` |

## statistics (12) — categoria ausente de TODA a documentação e das aulas

| Tipo | Rótulo | Pegadinha declarada na própria descrição |
|---|---|---|
| `stat_abc` | Curva ABC (Pareto) | Classes A/B/C por concentração acumulada. **Cita "margem" como coluna de valor típica** |
| `stat_rfm` | Segmentação RFM | Recency/Frequency/Monetary em quantis, com rótulos de segmento |
| `stat_anomaly` | Detecção de Anomalias | Responde **duas** perguntas distintas: valor atípico (z-score, z robusto, IQR) vs. mudança de patamar (cusum, ewma, exigem `date_column`). Base **retrospectiva** — não é controle online |
| `stat_forecast` | Previsão de Demanda | Holt-Winters, Croston (giro lento) ou média móvel, com intervalo |
| `stat_reorder` | Ponto de Pedido / Estoque de Segurança | **Informar `date_column`** ou períodos sem movimento não contam como zero e a média sai inflada |
| `stat_cohort` | Coorte / Retenção | Saída em formato longo |
| `stat_correlation` | Correlação | Pearson ou Spearman. **Não implica causalidade** |
| `stat_market_basket` | Cesta de Compras | Pares com support, confidence, lift |
| `stat_transition` | Matriz de Transição | **Entrada tem de ser snapshot por período.** Com eventos, devolve número plausível e errado, sem erro |
| `stat_drift` | Desvio de Distribuição | PSI com classificação (<0,10 estável; 0,10–0,25 atenção; >0,25 mudou). Duas populações saem da **mesma** entrada |
| `stat_woe` | Poder de Discriminação | WoE por faixa + IV por variável. **IV > 0,5 = suspeita de vazamento.** Exige desfecho já observado |
| `stat_discrimination` | Qualidade de Score | KS, Gini, AUC + tabela por decis. Avalia score existente; não cria nem recalibra |

## logic (6)

| Tipo | Rótulo | Observação |
|---|---|---|
| `if_node` | IF (Row Partition) | Particiona em `true`/`false`. Em **gate mode** (sem upstream tabular) ativa um único ramo |
| `switch_node` | Switch (Partição por Valor) | N ramos por valor de campo; também tem gate mode |
| `loop` | Loop (Iteração) | Subfluxo ou corpo inline por item. Sequencial/paralelo, **políticas de erro e limite de iterações** |
| `sync` | Sincronizar (Aguardar Todos) | Convergência de ramos paralelos; expõe `branch_refs` |
| `code` | Código Python | Docker efêmero, Python 3.12, **sem rede**, FS read-only. Recebe `data`/`connection`/`source_table`, pandas/numpy/polars. Deve atribuir a `result` |

## database (6)

| Tipo | Rótulo | Observação |
|---|---|---|
| `bulk_insert` | Bulk Insert | Estratégias `append_fast`, `append_safe`, `upsert`, `insert_if_not_exists` |
| `composite_insert` | Inserção Composta | **Cascata header + filhos em UMA transação, propagando FK via RETURNING** |
| `sql_script` | Script SQL | SELECT/INSERT/UPDATE/DELETE/DDL com parâmetros nomeados. Modos `query`, `execute`, `execute_many` |
| `loadNode` | Carga SQL (Load) | `append`, `replace` (TRUNCATE+INSERT), `merge` (UPSERT por chave) |
| `truncate_table` | Truncar/Deletar Tabela | TRUNCATE ou DELETE com WHERE opcional; repassa o upstream intacto |
| `internal_data_write` | Base de Dados Interna (Escrita) | `insert`, `upsert`, `update`, `delete`, `replace` |

## output (5)

| Tipo | Rótulo | Observação |
|---|---|---|
| `csv_export` | Exportar CSV | Delimitador, encoding, BOM UTF-8 para Excel |
| `xlsx_export` | Exportar XLSX | Teto de 1.048.576 linhas (limite do Excel) |
| `pdf_report` | Relatório PDF | KPIs, gráficos, tabelas. **Não agrega — some/agrupe num nó anterior.** Faz passthrough |
| `google_sheets_output` | Google Sheets (Gravar) | `append` ou `overwrite` |

## subflow (3)

| Tipo | Rótulo | Observação |
|---|---|---|
| `workflow_input` | Entrada do Sub-workflow | Expõe os parâmetros do `input_schema` |
| `workflow_output` | Saída do Sub-workflow | Empacota o resultado para o `call_workflow` pai |
| `call_workflow` | Chamar Sub-workflow | Roda em **isolamento** — contexto próprio, não herda upstream do pai |

---

## Leituras relevantes para o piloto de margem

1. **A consolidação de nós NÃO aconteceu como a doc previa.** `mapper`, `filter`, `sort`,
   `sample`, `record_id`, `union`, `pivot`, `unpivot`, `text_to_rows` **continuam existindo
   como nós separados**. Só `combine` foi criado, absorvendo `join` (marcado legado) e `union`.
   Não há nó `transform` nem `aggregate`. **A decisão de usar o nome legado como chave primária
   da base de conhecimento está validada.**
2. **`if_node` em gate mode viabiliza o dry-run** — variável booleana + gate desviando do nó de
   escrita para um de registro. É a primitiva que faltava (L2).
3. **`aggregator` (COUNT) + `if_node` viabilizam o disjuntor** de §5.4 (L5).
4. **`loop` tem "políticas de erro e limite de iterações"** — segundo mecanismo de contenção.
5. **`composite_insert` é feito para pedido + itens**: cascata header/filhos em uma transação
   com FK propagada. Não aparece em nenhuma aula.
6. **`stat_abc` é o primeiro lugar em todo o acervo onde "margem" aparece.** Não resolve L4
   (continua faltando saber onde a margem está no ERP), mas indica que a plataforma tem
   vocabulário de negócio para isso.
7. **12 nós de estatística não existem em nenhuma aula nem documento.** É a maior lacuna de
   cobertura do material — e vários são diretamente aplicáveis (curva ABC por margem, ponto de
   pedido, previsão de demanda).
8. **`sql_database` tem leitura particionada paralela**; `sql_script` tem `execute_many`
   iterando sobre o upstream. Ambos relevantes para volume.
