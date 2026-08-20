# Lacunas

Ordenado por **impacto no caso de uso alvo**: automação de ajuste de preço a partir de pedidos
(`fluxos/vendas/job-pedido-ajuste-margem`).

Semeado na **Fase 0** (2026-08-20). A Fase 1 expande isto por lote; a Fase 2 fecha o que o MCP
conseguir confirmar.

---

## Bloqueadores do piloto

### L1 — Execução concorrente do mesmo fluxo agendado
**Impacto: alto.** O piloto roda a cada 5 minutos. Se uma execução demorar 6, o que acontece?
A doc de cron (`guias-de-uso/nos/agendamento-(cron).md` § "Limites e guardrails") **não trata
disso** — nada sobre `max_instances`, coalescing ou misfire do APScheduler. Só afirma que dois
workflows *distintos* no mesmo horário não conflitam, o que é outra pergunta.
**O que falta descobrir:** a plataforma serializa execuções do mesmo workflow, ou permite
sobreposição? Se permite, existe primitiva de trava?
**Como resolver:** pergunta para o suporte Viasoft. Não é inferível.

### L2 — Não existe primitiva de dry-run
**Impacto: alto.** O plano (§5.4) exige dry-run como modo padrão para o fluxo que escreve
preço. Não há recurso equivalente na plataforma. O toggle Teste/Produção limita a 500 linhas,
mas **não** suprime escrita — é limite de volume, não de efeito.
**PRIMITIVA CONFIRMADA (Fase 2, 2026-08-20):** além do gate mode, `if_node` tem
**`decision_mode`**: `per_row` (padrão, reparte cada linha entre V/F) e **`single`** — *"avalia SÓ
a 1ª linha e envia a tabela inteira por um único ramo (decisão de lote)"*.

`decision_mode: single` **é** o dry-run: uma condição sobre `{{vars.dry_run}}` manda o lote inteiro
para o ramo de registro ou para o ramo de escrita.
**Rebaixada para: baixo.** A peça existe e é adequada. Falta só exercitar na Fase 5.

### L3 — ~~Watermark e carga incremental~~ RESOLVIDA — **é NATIVO** (2026-08-20, Fase 2)
**Status: fechada pelo MCP.** O termo tem zero ocorrências em 380 KB de transcrição e 40
documentos — mas **o recurso existe no nó**, só não é chamado de watermark.

`describe_node('sql_database')` `[CONFIRMADO-MCP]`:

| Parâmetro | O que faz |
|---|---|
| `incremental_column` | Coluna de corte (data, timestamp ou sequencial crescente) |
| `initial_value` | De onde começar na **primeira** execução. Vazio = lê tudo |
| `reprocess_window` | **Recuo aplicado ao marco a cada execução** — dias se data/hora, unidades se sequencial. *"Existe para pegar lançamento retroativo"* |

Os nomes `incremental_from`, `shift_p_lo` e `shift_p_hi` são **reservados pelo motor** e recusados
como nome de parâmetro — o marco é gerenciado internamente.

**`reprocess_window` é exatamente a "margem de sobreposição" que §5.1 do plano manda somar à
mão.** Não é preciso construir `sub-obter-watermark` nem guardar marco em base interna.

> ⚠️ Armadilha declarada: `incremental_column` **precisa de índice na origem** — *"sem ele o
> incremental vira varredura completa e o ganho some"*. Verificar o índice na tabela de pedidos
> do ERP antes de confiar.

`[LACUNA remanescente]` Onde o marco é persistido, e o que acontece com ele quando a execução
falha no meio. Isso decide se §5.1 ainda precisa de algo manual.

### L4 — Margem: onde está cadastrada no ERP
**Impacto: alto.** O termo "margem" **não aparece em nenhuma fonte** — nem aula, nem doc. É
conhecimento exclusivo do ERP, não da plataforma.
**Agravante:** com o módulo 4 reposto, o acervo está completo — **380 KB de transcrição e
40 documentos — e o termo "margem" continua com zero ocorrências.** Não é lacuna de
material faltante; é conhecimento que só existe no seu ERP.

**O que falta descobrir (responder na Fase 5):** tabela e campo da margem; se é por item,
produto, categoria ou cliente; o que caracteriza "pedido novo"; qual campo de data usar como
marca d'água; se há histórico de preço.

### L5 — Teto de variação e disjuntor
**Impacto: alto.** §5.4 pede limite de variação percentual por item e limite de itens
alterados por execução. Nenhum dos dois existe como recurso.
**PRIMITIVAS CONFIRMADAS (Fase 2, 2026-08-20):** teto de variação via `filter` sobre coluna
calculada em `math` — que **cria** coluna, preservando o valor original. Disjuntor via
`aggregator` (COUNT) + `if_node` com **`decision_mode: single`**, que é decisão de lote: exatamente
a semântica de disjuntor. E `loop` tem políticas de erro e limite de iterações.
**Rebaixada para: médio.** As peças existem. Segue sem nó explícito de "falhar/abortar" — L10.

---

## Importantes, não bloqueadores

### L6 — Idempotência além do UPSERT
**Impacto: médio-alto.** Confirmado que existe `UPSERT` / "atualizar por chave" /
`insert_if_not_exists`. Não confirmado como evitar **oscilação** (o fluxo corrigir de novo o
que já corrigiu). O plano (§5.2) pede regra convergente.
Reforço independente de que isso importa: webhook em modo `immediately` é **at-least-once** e
pode executar duas vezes (`guias-de-uso/nos/webhook-(gatilho).md`).

### L7 — Nós de banco de dados não têm página de documentação
**Impacto: médio-alto.** `Introducao/conceitos.md` § "Nó" promete "o catálogo completo está na
seção Nós", e `guias-de-uso/nos/` **não contém** `SELECT`, `SQL Script`, `Inserção em Massa`,
`Limpar Tabela` nem `Destino SQL`. Para esses cinco a **aula é a única fonte**, o que os deixa
travados em `[UI-OBSERVADA]` — e são justamente os nós que o piloto usa para escrever.

### L8 — ~~Transcrições do módulo 4 estão vazias~~ RESOLVIDA (2026-08-20)
**Status: fechada.** Os 6 arquivos foram repostos: **2.122 linhas / 170 KB**, formato
consistente com o resto do acervo (593 marcações `(AÇÃO:)`, zero timestamps).

**Ressalva importante:** o módulo 4 é um caso de **migração de cadastro de itens** a partir do
banco de um concorrente (ConstruShow), com uso do copiloto para minerar o schema. **Não é o
caso de uso de preço/margem** — o termo "preço" aparece 1 vez e "margem" nenhuma. Ele fecha a
lacuna de "caso ponta a ponta com inserção real", mas **não** contribui para L4.

### L9 — Hierarquia divergente entre três documentos
**Impacto: médio.** `Introducao/conceitos.md` diz Organização → Espaço → Grupo econômico →
Projeto. `documentacao-tecnica/controle-de-acesso.md` diz Organização → Workspace → Projeto e
afirma que Cliente **não** é nível de acesso. `entidade-cliente.md` traz uma terceira variante.
**Leitura provável (a confirmar):** são hierarquias de propósitos diferentes — organização de
trabalho vs. permissão. Resolver no lote 2, antes que contamine o glossário.

### L10 — Como abortar/falhar um fluxo deliberadamente
**Impacto: médio.** Pré-requisito de L5. **`dead_letter` NÃO é um tipo de nó** — `list_nodes`
não o devolve, apesar de `conceitos.md` e `exportar-e-importar.md` citarem. Dead-letter é
recurso de **execução**, lido via `ler_rejeicoes` (grupos por causa, amostras, campos sensíveis
mascarados). Segue sem resposta como **provocar** falha deliberada. Candidatos a investigar com
`describe_node`: políticas de erro do `loop`, e `sql_script` em modo `execute`.

### L11 — ~~Catálogo de tipos de variável~~ RESOLVIDA, com pegadinha (2026-08-20)
**Status: fechada pelo MCP.** `pending_set_variables` declara os 10 tipos: `string`, `number`,
`integer`, `boolean`, `object`, `array`, `table_reference`, `connection`, `file_upload`,
`secret`. Casa em número com os 10 vistos na UI (`m3-parte-1`), com uma troca: a UI mostra
"Formulário", o MCP expõe `table_reference`.

**⚠️ PEGADINHA:** `set_workflow_variables` (escrita **direta**) aceita **só 6** tipos — `string`,
`number`, `integer`, `boolean`, `object`, `array`. Ficam de fora justamente
`connection`, `file_upload`, `secret` e `table_reference`. Usar a ferramenta direta em vez da
build session **perde a capacidade de declarar variável de conexão ou de segredo** — que é
exatamente o que o piloto precisa. Mais um motivo para preferir `pending_*`.

Detalhes confirmados: `connection` exige `connection_type` (`postgres`/`mysql`/`sqlserver`/
`oracle`/`mongodb`); `file_upload` tem `accepted_extensions` e a instrução explícita de **nunca
incluir `.xls`**; `table_reference` declara `columns` com nome, tipo e obrigatoriedade.

### L12 — ~~Consolidação de nós: quando~~ RESOLVIDA (2026-08-20)
**Status: fechada pelo MCP.** A consolidação **não aconteceu como a doc previa**. `mapper`,
`filter`, `sort`, `sample`, `record_id`, `union`, `pivot`, `unpivot` e `text_to_rows` seguem
existindo como nós separados. Não há nó `transform` nem `aggregate`. Só `combine` foi criado,
absorvendo `join` — cuja descrição diz "Versão legada — prefira o nó `combine`".
**A decisão de usar o nome legado como chave primária está validada.** O único par
legado→sucessor real a anotar é `join` → `combine (mode='join')`. Ver `divergencias.md` D-MCP-3.

### L13 — ~~MCP não permite criar nem editar fluxo~~ **ESTAVA ERRADA** (2026-08-20)
**Status: refutada pelo MCP.** Eu havia registrado, com base só na doc, que provavelmente não
houvesse ferramenta de criação/edição. **O MCP expõe 48 ferramentas, não 15**, e entre elas
`create_workflow`, `create_project`, `add_node`, `add_edge`, `remove_node`, `remove_edge`,
`update_node_config`, `set_workflow_variables`, `criar_base_interna`, `escrever_documentacao`,
mais a família `pending_*` inteira (build session com aprovação em lote).
**Consequência:** o Claude Code pode **construir** fluxo, não apenas desenhar — muda a
expectativa da Fase 5. O caminho indicado é a build session `pending_*`, não as ferramentas
diretas. Ver `divergencias.md` D-MCP-1 e D-MCP-2.

**Lição de método:** a doc oficial estava 4 meses defasada e me levou a uma conclusão errada
sobre capacidade da plataforma. `[CONFIRMADO-DOC]` não substitui `[CONFIRMADO-MCP]` quando a
pergunta é "o que existe hoje".

---

## Menores

### L19 — 12 nós de estatística sem cobertura nenhuma no material
**Impacto: médio-alto.** A categoria `statistics` (`stat_abc`, `stat_rfm`, `stat_anomaly`,
`stat_forecast`, `stat_reorder`, `stat_cohort`, `stat_correlation`, `stat_market_basket`,
`stat_transition`, `stat_drift`, `stat_woe`, `stat_discrimination`) **não aparece em nenhuma
aula nem documento**. É a maior lacuna de cobertura do acervo.

Relevância direta: `stat_abc` classifica em A/B/C por concentração acumulada e **cita "margem"
como coluna de valor típica** — é a primeira e única ocorrência de "margem" em todo o material,
incluindo o MCP. `stat_reorder` e `stat_forecast` são candidatos a rotinas futuras.

Várias descrições já trazem pegadinha declarada (ex.: `stat_transition` exige snapshot por
período e "devolve um número plausível e errado, sem erro nenhum" se receber eventos).
**Ação:** `describe_node` em cada um, na Fase 1 ou 2.

### L20 — ~~Qual workspace a API Key alcança~~ RESOLVIDA (2026-08-20, Fase 2)
Workspace **`fd9cff30-2988-48db-a19b-9faba7ea06c4`**, com **apenas 2 conexões Oracle** e nada
mais: zero projetos, fluxos, webhooks e bases internas. Não é o `Treinamento` das aulas.

| Nome | Tipo | Host | Porta | Banco | Usuário | Pública |
|---|---|---|---|---|---|---|
| Viasuper Padrão - Gabriel | oracle | 192.168.90.218 | 30200 | ORCL | VIASOFTMERC | Sim |
| Viasuper Titan - Gabriel | oracle | 192.168.90.218 | 30100 | ORCL | VIASOFTMERC | Sim |

**Consequência:** o piloto começa em ambiente limpo. Não há fluxo existente de onde reaproveitar
padrão, e **não há base interna** — o que exigiria `criar_base_interna` (escrita) se a auditoria
fosse por ali. Ver L28.

### L28 — Onde gravar a trilha de auditoria ⚠️
**Impacto: ALTO.** O plano (§5, item 5) pede registrar antes/depois de cada alteração de preço.
Por natureza é **histórico que cresce sem parar**. Não há destino óbvio:

- **Base interna:** teto duro de **200.000 linhas** (`INTERNAL_DATA_MAX_ROWS`), e o contrato do
  `internal_data_write` é explícito — *"Ela é para CADASTRO"*. Pior: a recusa por volume **só
  aparece na execução, depois de o usuário já ter aprovado o fluxo**.
- **O que o próprio contrato recomenda não existe:** o mesmo texto diz que para histórico *"o
  destino certo é o nó **'Gravar em Conjunto de Dados'** (parquet, lido como tabela pelos
  fluxos)"*. **Esse nó não aparece em `list_nodes`.** Verifiquei com busca por `conjunto` e por
  `parquet dataset gravar` — nada.
- **Tabela própria no Oracle** via `bulk_insert` é a alternativa que sobra, mas cria objeto no
  banco do ERP.

**O que falta descobrir:** o nó de Conjunto de Dados existe na interface? Se sim, por que
`list_nodes` o omite? Se não, a auditoria vai para tabela dedicada no Oracle?
**É decisão de arquitetura do piloto, não detalhe.**

### L21 — Nós de banco: contrato real de config
**Impacto: médio-alto.** `describe_node` existe e devolve "contrato completo: campos de
configuração, transforms e exemplos". Isso **contorna L7** — os nós de banco não têm página de
doc, mas têm contrato consultável pelo MCP, que é fonte melhor que a aula.
**Ação:** rodar `describe_node` em `sql_database`, `sql_script`, `bulk_insert`,
`composite_insert`, `loadNode`, `truncate_table`, `internal_data_write` no lote 6 da Fase 1.

---

## Menores adicionais

### L22 — Existe um gatilho "Monitorar Mudanças"? (pode dispensar o watermark)
**Impacto: ALTO.** `Introducao/conceitos.md § Nó` `[CONFIRMADO-DOC]` lista **"Monitorar
Mudanças"** entre os gatilhos. `m1:65` `[UI-OBSERVADA]` mostra **"Monitoramento"** na biblioteca
de gatilhos. `m3-2A` cita **"Monitorar tabelas"**, dizendo que tem "aula específica". Mas
`list_nodes` `[CONFIRMADO-MCP]` **não devolve nenhum dos três** — os únicos `trigger` são
`manual`, `webhook` e `cron`.

**Por que é alto impacto:** se existe um gatilho que reage a mudança em tabela, o piloto pode
não precisar de polling a cada 5 minutos **nem de marca d'água** — o que reescreveria §5.1 e §5.3
do plano-mestre e eliminaria L1 (execução concorrente) e L3 (watermark) de uma vez.

**O que falta descobrir:** o nó existe hoje? Se sim, por que `list_nodes` o omite — whitelist
`allowed_tools` da chave, feature flag, ou versão? Se existe, como detecta mudança (trigger de
banco? polling interno? CDC?) e qual a granularidade.

### L23 — Nós citados na doc que o MCP não devolve
**Impacto: médio.** Além de L22, `conceitos.md § Nó` cita nós ausentes de `list_nodes`:
**Dead Letter** (categoria Banco de Dados), **Gmail (enviar e-mail)** e **WhatsApp via Z-API**
(categoria Integrações), **Analista IA** e **Decisão IA** (categoria IA), **Nota** e **Grupo**
(categoria Outros). `visao-geral.md` reforça e-mail e WhatsApp ("mandar por e-mail ou WhatsApp ao
final do fluxo").

Além disso a própria descrição de `list_nodes` `[CONFIRMADO-MCP]` contradiz a doc:
*"Integrações não são uma categoria: procure pelo NOME do serviço"*.

**Leitura parcial:** "Nota" e "Grupo" são elementos de organização do canvas e provavelmente não
são nós de engine; "Dead Letter" já se sabe que é recurso de execução, não nó (ver L10). Restam
sem explicação **Gmail, WhatsApp e os dois nós de IA**.
**Relevância para o piloto:** notificação de falha (`sub-notificar-falha` em `_compartilhado/`)
depende de saber se há nó de e-mail ou WhatsApp.

### L24 — Exportação: JSON ou YAML?
**Impacto: médio.** `m1:58` `[VÍDEO]` diz que exporta e importa **"através de JSON"**.
`guias-de-uso/exportar-e-importar.md` `[CONFIRMADO-DOC]` diz que há 4 formatos e **round-trip só
em YAML**, sendo o "Exportar JSON (canvas)" um *"formato legado"* e SQL/Python somente leitura.
**Registrado nos dois lados.** Falta confirmar o que o botão da interface faz hoje — decide o
formato de versionamento do repositório (Fase 3).

### L25 — Mapper: dois recursos da UI sem equivalente no contrato
**Impacto: baixo.** A UI do Mapeamento oferece **"De Para"** (com Adicionar equivalência +
Fallback) e **"Padrão"** (valor default se nulo/vazio) `[UI-OBSERVADA]` `m3-3A`, que não constam
na lista de transforms do MCP. Provável que compilem para `CASE WHEN` e `coalesce`, mas
**não confirmado**.
Relacionado: em `m3-3A` uma tentativa de usar `COALESCE` no modo Expressão **falhou** e exigiu o
copiloto para corrigir — apesar de `coalesce` **constar** nos transforms do MCP. Falta descobrir
a diferença entre o modo Expressão da UI e a `expression` do executor.

### L26 — `conceitos.md` tem frase truncada sobre o que vive num Espaço
**Impacto: baixo.** `conceitos.md § Espaço` termina em *"é nele que vivem"* — a enumeração
**não existe** no documento. Pelas aulas, ao menos: conexões, bases internas, arquivos, modelos
de entrada, formulários, fluxos-template, nós personalizados e grupos econômicos. Falta a lista
canônica.

### L14 — Relay é necessário para o Oracle do ERP? *(reenquadrada — escopo Oracle)*
**Impacto: médio-alto** (era menor quando o escopo incluía SQL Server).
Não documentado se é obrigatório. O relay é agnóstico de banco (só TCP), e `deploy-firebird.md`
mostra acesso direto sem relay quando o backend está na mesma rede.

**Mudança de leitura com o escopo Oracle:** as funções de **borda** (extract/apply em Parquet)
têm dialeto **declaradamente só Oracle** (`apply_dialect: oracle`, `extract_dialect: oracle`,
"Escopo Fase 1: só Oracle"). Se o ERP é Oracle, **os recursos avançados de borda se aplicam** —
deixam de ser curiosidade e passam a ser possibilidade real de desempenho.
**RESPONDIDO EM PARTE (Fase 2, 2026-08-20):** `test_connection` retornou **SUCESSO** nas duas
conexões Oracle, e nenhuma indica relay nos metadados. Para essas conexões, o Oracle em
`192.168.90.218` é alcançável **direto — relay não é necessário**.
`[LACUNA remanescente]` As duas apontam para o mesmo host, provavelmente desenvolvimento. Falta
confirmar o Oracle de **produção**. E as flags `RELAY_EXTRACT_ENABLED` /
`RELAY_APPLY_ROW_TOLERANCE_ENABLED` / `UPSERT_STAGING_MERGE_ENABLED` — todas **OFF por default** —
seguem sem verificação neste ambiente.

### L27 — Bug conhecido de Oracle + dlt
**Impacto: médio-alto** (novo, por causa do escopo Oracle).
`planejamento-dlt-(extracao-insercao).md §1.4` `[CONFIRMADO-DOC]` registra **bug conhecido de
Oracle com dlt**, e diz que **Oracle e Firebird usam fallback SQLAlchemy** em vez do caminho
padronizado.
**Por que importa:** o piloto lê e escreve Oracle. Se a extração cai em fallback, o ganho de
leitura particionada paralela do `sql_database` pode não valer para Oracle.
**O que falta descobrir:** qual é o bug, se já foi corrigido, e se a leitura particionada
funciona contra Oracle na prática.

### L15 — ~~Sintaxe cron livre~~ RESOLVIDA em parte (2026-08-20, Fase 2)
`describe_node('cron')` `[CONFIRMADO-MCP]`: o nó tem **um único parâmetro** — `cron_expression`
(string, obrigatório), exemplo `'0 8 * * 1-5'`. **Sem piso e sem lista de valores permitidos.**
O piso de 5 minutos é da **interface**, não do contrato.
`[LACUNA remanescente]` Se o backend valida o intervalo mínimo ao salvar. **Não testável sem
operação de escrita** — ver `divergencias.md §4.2`. Irrelevante para o piloto, que pede 5 min.

### L16 — Referência de API pública
Endpoints aparecem espalhados por vários documentos (`/workflows/{id}/execute`,
`/workflows/import`, `/workflows/{id}/clone`…), mas **não existe** página de referência,
OpenAPI, versionamento ou contrato estável.

### L17 — Seis documentos técnicos são proposta, não plataforma
`arquitetura-hibrida-(cloud-runner).md` ("Status: Proposta arquitetural"),
`upsert-rapido-staging-+-merge-set-based-(design).md`, `toleracia-a-erro-por-linha-na-borda-design.md`,
`extracao-na-borda-design.md`, `planejamento-dlt-(extracao-insercao).md` (flags OFF por
default, E2E não exercido) e `editor-de-documentacao-(keystatic).md` ("não funciona hoje").
Nunca marcar como `[CONFIRMADO-DOC]`.

### L18 — Migração de permissões pendente
`controle-de-acesso.md` registra a migração `c7d8e9f0a1b2` como **destrutiva e ainda não
aplicada**, exigindo re-provisionar Observadores. O modelo de papéis documentado pode não ser
o que está em produção hoje.
