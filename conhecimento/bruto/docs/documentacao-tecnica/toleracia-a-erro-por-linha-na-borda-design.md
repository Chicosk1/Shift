---
title: Tolerância a erro por linha na borda — Design
---

> Épica **Tolerância a erro por linha na borda** (CIAG-277..280), eixo *robustez*
> da épica "Relay aplicador na borda" (CIAG-254..262). Este documento nasceu na
> **T1 (CIAG-277)** — o desenho — e foi mantido atualizado conforme a
> implementação: **T2** (`LOG ERRORS`, Go), **T3** (bisseção, Go) e **T4**
> (dead-letter na nuvem + rollout).
>
> **Status (07/jul/2026): épica implementada (T1–T4).** Nasce **OFF** pela flag
> `RELAY_APPLY_ROW_TOLERANCE_ENABLED`. ⚠️ **E2E real (Oracle + relay + OCI) não
> exercido** — coberto por testes unitários (geração de SQL, bisseção, writer do
> Parquet de rejeições, consumo e anexo do ramo `on_error`). Rollout na §8.

## 1. Problema

Hoje, quando uma carga aplicada na borda esbarra num **registro problemático**
(ex.: uma linha que fere `IDX_UNQ_PESSOADOCEND_TIPOEND` com `ORA-00001`), **a
carga inteira falha e nada é gravado** — mesmo que só 1 linha entre 400 mil
esteja errada. A causa é direta: com a épica CIAG-263..267 o upsert na borda
virou um **MERGE set-based atômico** (`applyViaStaging` em
[`oracle.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/oracle.go)), e o INSERT do
append é **array binding por lote** (`buildInsertSQL`) — em ambos, **uma** linha
ruim aborta o statement inteiro. O caminho antigo pelo túnel (`load_service`)
fazia MERGE **linha a linha** dentro de um `begin_nested()` por linha, então
**separava** a linha ruim (dead-letter, via `RejectedRow`) e seguia.

Esta épica traz essa tolerância de volta para a borda: **gravar tudo que dá,
separar só os registros com problema, e mostrar quais e por quê**.

> **Conserto de fundo é ortogonal.** O erro concreto que motivou a épica
> (`ORA-00001` no teste de fogo de 07/jul) tem causa própria: `merge_keys`
> desalinhadas de `(IDPESS,TIPOEND)` + reuso de `IDEND`. Alinhar isso faz **não
> haver** linha ruim. Esta épica é para quando **há** uma linha genuinamente ruim
> — não substitui o alinhamento das chaves. Ver
> [`edge-apply-dedup-unique-columns-fix`].

## 2. Onde encaixa (os três caminhos de aplicação)

O aplicador Go tem hoje **três** caminhos, todos em
[`oracle.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/oracle.go). A tolerância a
erro precisa cobrir cada um — a estratégia difere porque a **atomicidade** e a
**idempotência** deles diferem:

| Caminho | Estratégia | SQL | Atomicidade hoje | Idempotente? |
| --- | --- | --- | --- | --- |
| **Append** | `append` | `buildInsertSQL` (array binding por lote) | 1 lote = 1 statement; falha aborta o lote | **Não** (reaplicar duplica) |
| **Append + RETURNING** | `append` + `returning_columns` | `buildInsertReturningSQL` (linha a linha) | 1 linha = 1 statement | **Não** |
| **Upsert set-based** | `upsert` / `insert_if_not_exists` | `buildStagingMergeSQL` (staging → 1 MERGE) | 1 MERGE de N linhas, tudo-ou-nada | **Sim** (MERGE por chave reconverge) |
| **Upsert por lote** (fallback do set-based) | idem | `buildMergeSQL` (MERGE por lote via `dual`) | 1 lote = 1 statement | **Sim** |

Essa tabela é a espinha dorsal das decisões das §5 e §6 (append precisa de
bookkeeping; upsert é seguro reprocessar).

## 3. Mecanismo primário — `LOG ERRORS` (Oracle)

O Oracle tem um mecanismo **nativo** de "registrar o erro da linha e seguir": o
**DML error logging**. Anexa-se `LOG ERRORS INTO <err_table> ('<tag>') REJECT LIMIT <n>` a um INSERT/MERGE; as linhas que violam constraint/tipo/`NOT NULL` vão
para a `<err_table>` **em vez de** derrubar o statement. É **uma passada** (sem
partir o lote) — por isso é o mecanismo primário **na retentativa**.

> **⚠️ Princípio central — tolerância REATIVA (fast-path-first).** O `LOG ERRORS`
> **não** é anexado ao primeiro apply. Ele (como a bisseção) só entra na
> **retentativa de um lote que falhou**. Motivo: `LOG ERRORS` sempre-ligado tem
> **custo no caminho feliz** — força o caminho convencional e desliga otimizações
> em bloco do INSERT/MERGE, mesmo quando **não há nenhuma** linha ruim; e no
> upsert ele nem captura o erro do `ON` (§3.4), o que seria custo sem benefício.
> O fluxo é: **(1) apply rápido, sem `LOG ERRORS`** (a velocidade de hoje); **(2)
> só se falhar por erro de dado**, reaplica o lote — reusando a staging já
> carregada — com `LOG ERRORS` (append) ou bisseção (upsert). Resultado:
> **caminho feliz = zero overhead**; o custo da tolerância só existe quando há
> linha ruim. Detalhe e critério de aceite na §3.5.

Este mecanismo já foi antecipado no design do upsert set-based
([`upsert-staging-merge-design.md`](https://shift.viasoftcloud.com.br/docs/technical/upsert-staging-merge-design.md) §5); aqui ele
é especificado por completo para os **dois** DMLs da borda.

### 3.1. SQL por caminho

**Append (INSERT):**

```
INSERT INTO <destino> (<C1..Cn>) VALUES (:1, …, :n)
  LOG ERRORS INTO <err_table> ('shift:<job_id>') REJECT LIMIT UNLIMITED;
```

**Upsert set-based (MERGE do staging):** anexa a mesma cláusula ao MERGE que
`buildStagingMergeSQL` já gera:

```
MERGE INTO <destino> t
USING ( … dedup por _SHIFT_SEQ … ) s
ON ( t."K1" = s."K1" AND … )
WHEN MATCHED THEN UPDATE SET …
WHEN NOT MATCHED THEN INSERT (…) VALUES (…)
LOG ERRORS INTO <err_table> ('shift:<job_id>') REJECT LIMIT UNLIMITED;
```

T2 implementa isto **parametrizando** `buildInsertSQL` e `buildStagingMergeSQL` /
`buildMergeSQL` — não é um gerador novo. A cláusula é emitida **apenas na
retentativa** de um lote que falhou (§3.5), quando a flag (§7) está ON e a
`<err_table>` foi garantida — **nunca no primeiro apply** (senão o caminho feliz
regride, §3.5).

### 3.2. Criação e higiene da err table

- **Nome determinístico por destino:** `ERR$_<hash(destino)>`, ≤ 30 chars
(limite de identificador do Oracle 11g/12c), no espelho de `stagingTableName`
em [`oracle_sql.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/oracle_sql.go). Um
nome estável permite reuso entre jobs e limpeza previsível.

- **Criada uma vez** via `DBMS_ERRLOG.CREATE_ERROR_LOG('<destino>', '<err_table>')`. A err table herda as colunas do destino (como `VARCHAR2(4000)`,
tolerante ao dado ofensor) + as colunas de controle `ORA_ERR_NUMBER$`,
`ORA_ERR_MESG$`, `ORA_ERR_ROWID$`, `ORA_ERR_OPTYP$`, `ORA_ERR_TAG$`.

- **Corrida de criação:** duas execuções criando a mesma err table → uma leva
`ORA-00955 (name is already used)`; tratamos como **sucesso** (a tabela
existe), idêntico ao `ensureStagingGTT`.

- **Higiene (crítico — dado de cliente):** a err table guarda **cópia das linhas
rejeitadas**, que é dado de cliente. Não pode acumular. Estratégia:

  1. **Isolamento por job:** cada job usa a tag `'shift:<job_id>'` na cláusula
`LOG ERRORS`, gravada em `ORA_ERR_TAG$`. A leitura das rejeições (§5) filtra
por essa tag — jobs concorrentes não se veem.

  2. **Limpeza no fim do job:** após ler as rejeições, o relay executa
`DELETE FROM <err_table> WHERE ORA_ERR_TAG$ = 'shift:<job_id>'` na **mesma
conexão**, antes de liberar. A err table é objeto persistente (como a GTT de
staging), mas seus **dados** não sobrevivem ao job.

  3. **Rede de segurança:** T4 documenta uma varredura de idade no cliente
(`DELETE … WHERE ORA_ERR_TAG$ LIKE 'shift:%'` para tags órfãs), espelhando o
`RELAY_APPLY_ARTIFACT_MAX_AGE_HOURS` do bucket.

### 3.3. Privilégio exigido

O usuário de **aplicação** do relay (o `apply_user` do destino no `relay.yaml`,
resolvido em [`executor.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/executor.go))
precisa de:

- `CREATE TABLE` (para o `DBMS_ERRLOG.CREATE_ERROR_LOG` — é DDL). O mesmo
privilégio já exigido pela GTT de staging, então **não é um requisito novo**
onde o upsert set-based já roda.

- `INSERT`/`SELECT`/`DELETE` na err table (implícito pela posse, já que ele a
cria).

Sem `CREATE TABLE`, a err table não pode ser criada → o mecanismo primário é
**indisponível** e a carga cai no **fallback** (§4). Detecção: `ORA-01031 (insufficient privileges)` na criação, mapeado para um sentinela
`errRowLoggingUnavailable` (espelho de `errStagingUnavailable`).

### 3.4. Restrições conhecidas do `LOG ERRORS` (validar em T2)

`LOG ERRORS` **não captura todos os erros** — o design assume estas limitações e
o fallback (§4) as cobre:

- **Violação de índice único usado no próprio `ON` do MERGE** pode **não** ser
desviada para a err table em algumas versões (o Oracle precisa da unicidade
para decidir MATCHED/NOT MATCHED). ⚠️ É **exatamente** o caso do `ORA-00001`
que motivou a épica — então para o upsert o **fallback por bisseção é o
caminho de garantia**, não um mero plano B.

- Erros de **constraints deferidas** disparam só no commit, fora do alcance do
`LOG ERRORS` do statement.

- Tipos LOB/LONG e `direct-path` têm ressalvas.

- `REJECT LIMIT` (§7): default `UNLIMITED` para "gravar tudo que dá", mas
configurável — atingir o limite **aborta** o statement (comportamento atual),
útil como circuit-breaker quando "quase tudo" está ruim.

### 3.5. Performance do caminho feliz (fast-path-first)

A tolerância é **reativa** para não custar nada quando não há erro:

1. **1º apply — rápido, SEM `LOG ERRORS`:** INSERT array-binding (append) ou
MERGE set-based do staging (upsert), exatamente como hoje.

2. **Sucesso** → fim. **Zero overhead** — o caminho feliz mantém a performance
atual da borda.

3. **Falha por erro de dado** (não conexão/cancelamento) → reaplica **o mesmo
lote / a mesma staging já carregada** (sem recarregar) com o mecanismo
tolerante: `LOG ERRORS` (append) ou bisseção (upsert, §4).

Trade-off assumido: quando **há** linha ruim, paga-se ~1 apply extra (o rápido
que falhou + a passada tolerante) — aceitável, pois o custo só aparece no caso
ruim. **Nunca** se paga no caminho feliz. A falha rápida é barata: o statement
aborta no primeiro registro ofensor, sem processar o lote todo.

> **Critério de aceite (T2/T4):** com a flag **ON** e **sem** nenhuma linha ruim,
> a carga na borda **não regride** perceptivelmente vs flag OFF — medir o mesmo
> dataset com e sem registros ruins. Se houver regressão no caso limpo, é sinal
> de que o `LOG ERRORS` está sendo anexado ao 1º apply: corrigir para reativo.

## 4. Fallback — bisseção adaptativa (T3)

Quando o `LOG ERRORS` é **indisponível** (sem `CREATE TABLE`), **desligado por
config**, ou **insuficiente** (o erro não foi desviado — caso do `ON` do MERGE),
o relay isola as linhas ruins **dividindo o lote**:

1. Aplica o lote inteiro (INSERT array-binding ou MERGE) numa transação.

2. **Sucesso** → commit, segue.

3. **Falha de dados** (não erro de conexão/cancelamento) → `rollback`, **parte o
lote ao meio** e recorre em cada metade.

4. **Lote de 1 linha que falha** = a linha ruim: captura `ORA_ERR_NUMBER$`/
mensagem via o erro do driver, registra como rejeição (§5) e **segue** com o
resto.

5. **Teto de passadas / profundidade** para não degenerar (ex.: se > X% do lote
é ruim, aborta com erro claro em vez de bissectar 400k linhas uma a uma — o
`REJECT_LIMIT` da §7 governa esse teto).

**Idempotência por estratégia (a decisão-chave — §6):**

- **Upsert (MERGE):** seguro. Cada sub-lote é um MERGE por chave, idempotente;
reprocessar uma metade já aplicada **reconverge**. A bisseção pode commitar
metades livremente.

- **Append (INSERT):** **exige bookkeeping**. Uma metade commitada e depois
reprocessada **duplica** linhas. A bisseção do append precisa registrar, por
sub-lote, o **intervalo de linhas já commitado** (offset+len sobre a ordem do
Parquet) e **nunca** reaplicar um intervalo commitado. Em caso de interrupção
no meio, o append **não retoma automaticamente** (mesma regra do
`resumeFromCheckpoint`: append não é idempotente → erro pedindo intervenção).

A bisseção é também o **caminho genérico para bancos futuros** (PG/MySQL/MSSQL),
que não têm o `LOG ERRORS` do Oracle.

## 5. Contrato do dead-letter da borda (rejeições → Shift)

O objetivo: as linhas rejeitadas na borda **voltam para o Shift** e aparecem na
execução como já aparecem no túnel — **nº de rejeitadas + amostra + motivo**,
alimentando o ramo `on_error`/dead-letter.

### 5.1. Reuso do caminho reverso (CIAG-276)

O canal relay → bucket → Shift **já existe**: é o do RETURNING
([`returning.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/returning.go) +
[`relay_apply_consume.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/relay_apply_consume.py)).
Reusamos a mesma mecânica, sem inventar transporte:

- O relay escreve um **artefato de rejeições** — `rejected.parquet` — em `JobDir`
([`rejected_writer.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/rejected_writer.go),
Parquet + zstd). Independente do `returned.parquet` da CIAG-276.

- O `Manifest` ([`manifest.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/manifest.go))
ganha um campo `Rejected *ReturnSpec` (reusa o **mesmo tipo** do `return`:
`key` + `upload.put_url`). O relay sobe o artefato via
[`executor.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/executor.go) **antes de
limpar** o `JobDir` — **não-terminal**: as linhas boas já foram aplicadas, então
falha no upload do dead-letter só loga (a contagem ainda aparece).

- O backend baixa e materializa, espelhando `materialize_returned_parquet` — um
novo `materialize_rejected_parquet`
([`relay_apply_consume.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/relay_apply_consume.py))
que produz o artefato do ramo `on_error` (`{node_id}_on_error`, a **mesma**
convenção que o `JsonlStreamer` usa no túnel, ver
[`bulk_insert.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/workflow/nodes/bulk_insert.py)).

- O `dispatch` ([`relay_apply_dispatch.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/relay_apply_dispatch.py))
anexa `branches["on_error"]` + `active_handles += on_error` e põe
`rejected_count` no resultado do nó → o `dynamic_runner` marca o nó **partial**
e a execução mostra a contagem; o nó `dead_letter` ligado ao handle `on_error`
persiste as linhas (repositório de dead-letter, CIAG-75/76) — **igual ao túnel**.

> Nota: o dead-letter da borda vale para **append e upsert** (o RETURNING é só
> append). Um job de upsert sobe **só** o `rejected.parquet`.

### 5.2. Schema do `rejected.parquet` e mapeamento para o dead-letter

> **⚠️ Correção vs. o rascunho da T1.** O desenho original previa `_SHIFT_SRN` +
> **JOIN** com o Parquet de carga (espelhando a CIAG-276). Isso **não se aplica**
> aqui: a err table do `LOG ERRORS` guarda cópias das **colunas do destino**, não
> o `_SHIFT_SRN` (que nunca é inserido). Então o `rejected.parquet` carrega **os
> próprios valores da linha ruim** (que o relay já tem — da err table ou da
> bisseção) + o motivo, **sem JOIN**. É mais simples e vale igual para LOG ERRORS
> e bisseção.

Schema (tudo texto — a err table é `VARCHAR2`; a bisseção lê os valores como texto):

| Coluna no `rejected.parquet` | Origem na borda | Vira no Shift (`on_error`) |
| --- | --- | --- |
| (colunas de dados do nó) | valores da linha ruim (err table / bisseção) | as próprias colunas → `payload` do dead-letter |
| `_SHIFT_ERR_MESG` | `ORA_ERR_MESG$` (LOG ERRORS) ou msg do driver (bisseção) | `_shift_rejection.error` → `RejectedRow.error` |
| `_SHIFT_ERR_CODE` | `ORA_ERR_NUMBER$` (ex.: `1` para `ORA-00001`) | `_shift_rejection.error_code` |

- **Recolhimento no consumo (CIAG-595):** `materialize_rejected_parquet` projeta
as colunas de dados como estão e recolhe `_SHIFT_ERR_MESG`/`_SHIFT_ERR_CODE`
numa **única coluna de controle** `_shift_rejection` (JSON) — o mesmo contrato
que o túnel escreve. Antes elas viravam marcadores soltos `_dead_letter_*` no
meio das colunas do cliente, o que contaminava o arquivo baixado e sobrescrevia
em silêncio uma coluna homônima. O vocabulário compartilhado (empacotar, ler e
tolerar o formato antigo) vive em `app/services/dead_letter_payload.py`. **A
borda não muda: o relay continua escrevendo `_SHIFT_ERR_*`** — não há binário
a redistribuir.

- **Contagem:** `rows_rejected` entra no `jobs.Result` do executor (ao lado de
`rows_applied`), e o `rows_applied` já é **linhas boas** (total − rejeitadas). O
`dispatch` promove isso a `rejected_count` no resultado do nó (CIAG-75/76).

- **Amostra:** a amostra por nó na tela de execução vem do **repositório de
dead-letter** (GROUP BY em `workflows.py`), alimentado pelo nó `dead_letter` a
partir do ramo `on_error` — o `rejected.parquet` traz **todas** as rejeitadas.

### 5.3. Fluxo fim a fim

```
relay: aplica (LOG ERRORS ou bisseção)
   → boas gravadas no destino
   → ruins → err table (tag do job) / capturadas na bisseção
   → lê rejeições, escreve rejected.parquet (valores da linha + motivo/código)
   → DELETE das rejeições da err table (higiene, §3.2)
   → sobe rejected.parquet pro bucket (put_url do manifesto) [não-terminal]
   → jobs.Result { rows_applied (boas), rows_rejected }
Shift: baixa rejected.parquet
   → materialize_rejected_parquet → {node_id}_on_error (controle em _shift_rejection)
   → dispatch anexa branch on_error + rejected_count no nó
   → nó dead_letter persiste (repositório) → execução: nº + amostra + motivo
```

## 6. Idempotência e segurança por estratégia (registro da decisão)

| Estratégia | Reprocessar é seguro? | Regra na tolerância a erro |
| --- | --- | --- |
| **Upsert / insert_if_not_exists** (MERGE) | **Sim** — por chave, reconverge | `LOG ERRORS` no MERGE; se insuficiente (caso do `ON`), bisseção pode commitar metades livremente. Sem bookkeeping. |
| **Append** (INSERT) | **Não** — reaplicar duplica | `LOG ERRORS` no INSERT em uma passada (preferido, sem risco de duplicar). Na **bisseção**, bookkeeping obrigatório dos intervalos commitados; **sem retomada automática** após interrupção (erro pedindo intervenção, como `resumeFromCheckpoint`). |

Esta é a mesma invariante que já governa a retomada por checkpoint em
`oracle.go` (`resumeFromCheckpoint`: MERGE reinicia do zero e reconverge; append
falha em vez de duplicar). A tolerância a erro **estende** essa invariante, não a
contradiz.

## 7. Flag de rollout e `REJECT_LIMIT`

Seguindo a convenção `_ENABLED` (default OFF) do backend
([`config.py`](https://shift.viasoftcloud.com.br/shift-backend/app/core/config.py), como
`RELAY_APPLY_RETURNING_ENABLED` / `RELAY_EXTRACT_ENABLED`):

- **`RELAY_APPLY_ROW_TOLERANCE_ENABLED: bool = False`** — porta mestre da épica.
Com OFF (default), o comportamento é **exatamente o de hoje**: uma linha ruim
aborta a carga (tudo-ou-nada). O gate viaja no **manifesto** (como
`use_staging_merge`): o backend só marca `row_tolerance=true` quando a flag
está ON e o destino é elegível; o relay não muda de comportamento sem o backend
mandar. Relays antigos ignoram o campo. **Mesmo com ON**, o 1º apply é o rápido
(sem `LOG ERRORS`); a tolerância entra só na retentativa de um lote que falhou
(§3.5) — o caminho feliz não paga nada.

- **`RELAY_APPLY_REJECT_LIMIT: int = -1`** — o `REJECT LIMIT` da cláusula `LOG ERRORS` e o teto da bisseção. `-1` = `UNLIMITED` ("grava tudo que dá,
independente de quantas ruins"). Um valor positivo `N` faz a carga **abortar**
se passar de `N` rejeições — circuit-breaker para "a carga inteira está errada"
(evita mascarar um mapeamento quebrado como "só rejeições"). O manifesto
carrega o valor efetivo para o relay.

Sem flag nova no relay além do que o manifesto carrega: o relay lê
`row_tolerance` + `reject_limit` do `ApplySpec`, como já lê `use_staging_merge`.

## 8. Escopo, ordem e o que T2–T4 herdam

- **Escopo Fase 1: só Oracle** (igual às épicas de borda). Bancos futuros usam a
bisseção genérica (§4) sem `LOG ERRORS`.

- **Reativo — fast-path-first (§3.5):** 1º apply rápido **sem** `LOG ERRORS`; a
tolerância entra **só na retentativa** de um lote que falhou → **caminho feliz
sem regressão** (critério de aceite medível em T2/T4). Na retentativa, `LOG ERRORS` (§3) é o mecanismo primário; a **bisseção** (§4) é fallback/garantia
(privilégio ausente, OFF, ou erro no `ON` do MERGE — o motivador da épica).

- **Contrato do dead-letter** (§5): reusa o canal reverso da CIAG-276
(`ReturnSpec` + `relay_apply_consume`) e o formato de rejeição da CIAG-75/76
(`RejectedRow` / `_shift_rejection` / ramo `on_error`).

- **Flag** (§7): `RELAY_APPLY_ROW_TOLERANCE_ENABLED` (OFF) + `RELAY_APPLY_REJECT_LIMIT`.

- **Idempotência** (§6): upsert livre; append com bookkeeping e sem retomada auto.

**Ordem de implementação:** T1 (este doc) → **T2** (`LOG ERRORS` em
`oracle_sql.go`/`oracle.go` + err table + higiene) e **T3** (bisseção) em
paralelo → **T4** (upload/consumo do `rejected.parquet` no backend, higiene das
`ERR$_*` no cliente, flag ON, doc de operação e E2E real).

### Requisitos de rollout (T4)

- **Privilégio** `CREATE TABLE` no `apply_user` do relay para a err table (o
mesmo do staging; sem ele → só bisseção). Combinar com o instalador/doc do
relay ([`shift-relay.iss`](https://shift.viasoftcloud.com.br/shift-connector/installer/shift-relay.iss)).

- **Higiene das `ERR$_*`** no Oracle do cliente (limpeza por tag no fim do job +
varredura de idade — §3.2).

- **Lifecycle Policy** do bucket cobrindo `rejected.parquet` (como já se exige
para `returned.parquet` da CIAG-276).

- **Flag ON** + relay reconstruído + E2E real (Oracle + relay + OCI) — não
exercitado nesta tarefa de design.

## 8.1 Paridade na via direta (conexão direta ao banco) — CIAG-283

A borda (CIAG-277..280) resolveu a tolerância por linha para cargas que passam
pelo relay. A **via direta** — quando o Shift alcança o banco sem relay e faz o
upsert por `staging + MERGE` (CIAG-264, flag `UPSERT_STAGING_MERGE_ENABLED`) —
ficara com o comportamento antigo: no erro do MERGE set-based, caía no
**row-by-row** (MERGE de 1 linha por linha), que **reprocessa o conjunto inteiro**
e é lentíssimo (um fluxo menor que uma carga de borda levou **>40 min** vs. **7
min** na borda). A **CIAG-283** iguala o *comportamento* (não o código) portando a
**bisseção** para a via direta, em
[`load_service.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/load_service.py)
(`_upsert_via_staging_merge`).

### O que foi portado (gêmeos Go → Python)

| Borda (Go, `row_tolerance.go`) | Via direta (Python, `load_service.py`) |
| --- | --- |
| `buildStagingMergeRangeSQL` | `_build_staging_merge_range_sql` |
| `buildReadStagingRowSQL` | `_build_read_staging_row_sql` |
| `bisectRange` | `_bisect_range` |
| `bisectStagingMerge` | `_bisect_staging_merge` |
| `rejectionFromStagingRow` | `_rejected_from_staging_row` (→ `RejectedRow`) |
| `rejectionCollector` + REJECT LIMIT | contagem de `rejected` + `UPSERT_REJECT_LIMIT` + `_RejectLimitExceeded` |

Fluxo: no caminho feliz, **um único MERGE** set-based (sem bisseção). Se ele
falha por linha(s) ruim(s), o `except` bisseciona **na mesma transação** sobre as
faixas de `_SHIFT_SEQ` da staging já carregada — ~O(log n) sobre a porção ruim,
mantendo a velocidade set-based no restante. As linhas isoladas viram
`RejectedRow` (dead-letter) e o nó termina **`partial`**. Só
`_StagingUnavailable` (GTT não pôde ser criada) cai no row-by-row antigo; o erro
por dado **não** cai mais nele.

### Divergências deliberadas (comportamento, não código idêntico)

- **`LOG ERRORS`/`DBMS_ERRLOG` não foi portado.** Na borda ele é o mecanismo
primário (bisseção é fallback); na via direta seria custo alto por pouco ganho —
e ele **não desvia exceção de trigger** (o caso motivador `ORA-01403`). A
bisseção sozinha dá a garantia. Coerência de *comportamento*.

- **SAVEPOINT (`begin_nested`) por faixa é obrigatório.** SQLAlchemy 2.0 invalida
a **transação inteira** a qualquer erro de statement; sem o SAVEPOINT, a falha
de uma faixa descartaria o staging (GTT `ON COMMIT DELETE ROWS`) e as faixas
boas. A borda (Go/`database/sql`) se virou com rollback em nível de statement.
A leitura da linha rejeitada (`_rejected_from_staging_row`) também roda sob
SAVEPOINT próprio.

- **REJECT LIMIT com default protetor.** `UPSERT_REJECT_LIMIT` (default **1000**;
`<0` = ilimitado) aborta com **causa provável** (trigger/FK) quando um caso
**sistêmico** (trigger que falha em quase toda linha) faria a bisseção
degenerar e dead-letterar em massa. Sem o teto, o custo vira ~O(n).

Fora de escopo (igual à borda): corrigir os erros de dado do ERP em si — FK
(`ORA-02291`, ordem de carga das tabelas-pai) e trigger (`ORA-01403`, regra do
ERP) são do fluxo/origem, não da plataforma.

Testes: [`test_upsert_staging_bisection_ciag283.py`](https://shift.viasoftcloud.com.br/shift-backend/tests/test_upsert_staging_bisection_ciag283.py)
(builder por faixa, leitura de linha, bisseção com executor fake — isola 1/2,
cobre sem duplicar, ~O(log n), caminho feliz = 1 chamada, REJECT LIMIT aborta).

## 9. Referências no código

- Aplicador Go: [`oracle.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/oracle.go)
(`applyViaStaging`, `resumeFromCheckpoint`, `errStagingUnavailable`),
[`oracle_sql.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/oracle_sql.go)
(`buildInsertSQL`, `buildStagingMergeSQL`, `stagingTableName`).

- Canal reverso (CIAG-276):
[`returning.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/returning.go)
(`returnWriter`, `returnedParquetName`),
[`manifest.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/manifest.go)
(`ReturnSpec`), [`executor.go`](https://shift.viasoftcloud.com.br/shift-connector/internal/apply/executor.go)
(upload + limpeza),
[`relay_apply_consume.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/relay_apply_consume.py)
(`materialize_returned_parquet`, JOIN por `_SHIFT_SRN`).

- Dead-letter/rejeição (CIAG-75/76):
[`load_service.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/load_service.py)
(`RejectedRow`),
[`bulk_insert.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/workflow/nodes/bulk_insert.py)
(ramo `on_error`, `_shift_rejection`),
[`dead_letter.py`](https://shift.viasoftcloud.com.br/shift-backend/app/services/workflow/nodes/dead_letter.py).

- Flags: [`config.py`](https://shift.viasoftcloud.com.br/shift-backend/app/core/config.py)
(`RELAY_APPLY_RETURNING_ENABLED`, `RELAY_EXTRACT_ENABLED` — convenção `_ENABLED`).

- Design par: [`upsert-staging-merge-design.md`](https://shift.viasoftcloud.com.br/docs/technical/upsert-staging-merge-design.md)
(§5 antecipa o `LOG ERRORS`), [`extracao-borda-design.md`](https://shift.viasoftcloud.com.br/docs/technical/extracao-borda-design.md).
