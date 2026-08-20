---
title: Upsert rápido — staging + MERGE set-based (design)
---

> **Status (06/jul/2026):** épica **implementada** (T0 design · T1 `load_service`
> Python direta+túnel · T2 aplicador Go da borda · T3 aviso de índice na UI).
> **T4 (CIAG-267): rollout = mantido OPT-IN — a flag `UPSERT_STAGING_MERGE_ENABLED`
> segue OFF por padrão.** Justificativa, benchmark e nota de operação na §9/§10.

## 1. Por que (o gargalo)

O upsert (`load_strategy="upsert"` / `insert_if_not_exists`) é hoje o **maior
gargalo de tempo** dos fluxos que gravam em Oracle. A causa é o protocolo
conversacional do Oracle combinado com o desenho **linha a linha** do caminho
atual:

| Frente | Caminho atual | Custo dominante |
| --- | --- | --- |
| **Direta** (`load_service`) | `_upsert_via_sqlalchemy`: **1 `MERGE INTO … USING (SELECT :c FROM dual)` por linha**, cada um dentro do seu `begin_nested()` (savepoint) | 1 round-trip + 1 savepoint **por linha** |
| **Túnel** (`load_service` via relay) | idêntico à direta — só troca o socket TCP | 1 round-trip por linha **atravessando a WAN** (~20–100 ms cada) → inviável em volume |
| **Borda** (aplicador Go) | `oracleApplier.Apply`: MERGE com **array binding** — 1 round-trip por **lote**, mas o servidor ainda executa **N MERGEs de 1 linha** (`USING (SELECT :1 FROM dual)`) | round-trip amortizado (já rápido: 32k linhas/s na LAN), mas **não é set-based** |

Referências no código:
`shift-backend/app/services/load_service.py` → `_upsert_via_sqlalchemy`,
`_build_merge_sql_for_dialect`;
`shift-connector/internal/apply/oracle_sql.go` → `buildMergeSQL`;
`shift-connector/internal/apply/oracle.go` → `oracleApplier.Apply`.

**A ideia:** trocar "N MERGEs de 1 linha" por **1 MERGE de N linhas**. Carregar
todas as linhas numa **área temporária** (staging) no próprio Oracle de destino e
rodar **um** `MERGE INTO destino USING staging ON (chaves)`, deixando o otimizador
do Oracle fazer o join em bloco (hash join, paralelizável) e tocar o índice do
destino **uma vez** em vez de por linha.

## 2. Contrato do MERGE set-based

O contrato hoje já existe nos dois lados (`buildMergeSQL` no Go,
`_build_merge_sql_for_dialect` no Python). A **única** mudança de contrato é a
cláusula `USING`: de um `SELECT … FROM dual` alimentado por bind, para uma
**tabela de staging**.

Dado o `column_mapping` (→ colunas `C1..Cn`) e os `merge_keys` (→ `K1..Km`):

```
MERGE INTO <destino> t
USING (
  SELECT <C1..Cn>
    FROM ( SELECT <C1..Cn>,
                  ROW_NUMBER() OVER (PARTITION BY <K1..Km>
                                     ORDER BY "_SHIFT_SEQ" DESC) rn
             FROM <staging> )
   WHERE rn = 1                       -- dedup: “última linha vence” (§4)
) s
ON ( t."K1" = s."K1" AND … AND t."Km" = s."Km" )
WHEN MATCHED THEN UPDATE SET t."Cx" = s."Cx", …   -- omitido se insert_if_not_exists
WHEN NOT MATCHED THEN INSERT (<C1..Cn>) VALUES (s."C1", …, s."Cn")
LOG ERRORS INTO <err_table> ('shift') REJECT LIMIT UNLIMITED;   -- erro-por-linha (§5)
```

Regras (idênticas ao gerador atual, portáveis entre Python e Go):

- **Colunas de update** = todas as colunas mapeadas **menos** as `merge_keys`.

- `insert_if_not_exists` ⇒ **omite** `WHEN MATCHED` (só insere as novas).

- Identificadores sempre com aspas duplas (`"COL"`) para preservar case e barrar
injeção; nome da tabela validado por regex antes de interpolar (já feito em
`validateTableName` / `_validate_table_name`).

- **Sem `RETURNING`** neste caminho — MERGE em Oracle não tem RETURNING portável.
Quando o nó pede `returning_columns`, cai no fallback row-by-row (§6), como já
acontece hoje.

> `T1`/`T2` implementam isto **parametrizando o `USING`** dos geradores que já
> existem — não é um gerador novo. O `SELECT … FROM dual` por linha continua vivo
> como caminho de fallback.

## 3. Mecanismo de staging (a decisão-chave)

Avaliamos os três candidatos da issue por **privilégio exigido**, **versão Oracle**,
**limpeza** e **concorrência entre execuções**:

| Mecanismo | Privilégio | Versão | Limpeza | Concorrência | DDL por execução |
| --- | --- | --- | --- | --- | --- |
| **A. GTT reusada** (Global Temporary Table) | `CREATE TABLE` **uma vez** (ou DBA pré-cria) | **Todas** (8i+) | Automática: linhas são por-sessão, sumem no commit (`ON COMMIT DELETE ROWS`) | **Isolada por sessão** por design — a MESMA GTT, dados separados por sessão | **Nenhum** após a 1ª criação |
| **B. PTT** (Private Temporary Table) | `CREATE PRIVATE TEMPORARY TABLE` (privilégio leve, dedicado) | **18c+** | Automática: dropada no commit (`ON COMMIT DROP DEFINITION`) | Nome escopado à sessão (prefixo `ORA$PTT_`) — sem colisão | `CREATE PRIVATE TEMPORARY TABLE` a cada execução, mas **não faz commit implícito** (PTT é isenta) |
| **C. Coleção `TABLE(:array)`** | Exige `CREATE TYPE` (é DDL) + 1 TYPE **por formato** de tabela | Todas | N/A (efêmera no bind) | OK | — |

**Decisão: A (GTT reusada) como primário; B (PTT) como upgrade automático em
18c+ quando falta `CREATE TABLE`; C descartada.**

Justificativa:

- **C sai:** promete "sem privilégio", mas um `CREATE TYPE … AS TABLE OF` é DDL e
exigiria **um TYPE por layout de coluna** — inviável para tabelas de cliente
arbitrárias. Além disso o binding de coleção do go-ora é fraco/ausente, então
**quebra o requisito "conceito compartilhável entre Python e Go"**. Fica só
como nota histórica.

- **A ganha o primário:** funciona em **todas** as versões Oracle que os clientes
rodam (11g/12c inclusive), é **isolada por sessão** (a concorrência sai de
graça: duas execuções na mesma GTT não se veem), e a **limpeza é automática**
(o commit final zera as linhas da sessão). O DDL acontece **uma única vez** por
formato de tabela — não a cada execução, então não há o problema do commit
implícito do DDL abortando a transação da carga.

- **B como degrau:** em Oracle 18c+, se o usuário do banco **não** tem
`CREATE TABLE` mas tem `CREATE PRIVATE TEMPORARY TABLE`, a PTT cobre o caso sem
footprint persistente (dropa sozinha). É criada por execução, mas PTT é isenta
de commit implícito, então não atrapalha.

### Como a GTT é materializada (A)

Nome **determinístico** por formato — `SHIFT_STG_<hash(destino+colunas)>` — e
criada **preguiçosamente uma vez**, herdando os tipos exatos do destino:

```
CREATE GLOBAL TEMPORARY TABLE SHIFT_STG_<hash> ON COMMIT DELETE ROWS AS
  SELECT <C1..Cn>, CAST(NULL AS NUMBER) AS "_SHIFT_SEQ"
    FROM <destino> WHERE 1=0;
```

- `AS SELECT … WHERE 1=0` copia os tipos **exatos** das colunas do destino (nada
de VARCHAR genérico que corromperia número/data no join).

- `"_SHIFT_SEQ"` é uma coluna sintética de ordem de chegada, preenchida no load
(sequencial por linha) — serve de critério de desempate determinístico do dedup
(§4: "última linha vence", igual à semântica atual row-by-row).

- **Corrida de criação:** duas execuções criando a mesma GTT ⇒ uma leva
`ORA-00955 (name is already used)`; tratamos como **sucesso** (a tabela existe,
é isso que queremos). Idempotente por natureza.

### Fluxo de uma carga (por chunk / por job)

1. Garante a GTT (cria-se-não-existe; sem custo após a 1ª vez).

2. **Carrega o staging** com array binding / `executemany` (Python) ou array
binding do go-ora (borda) — poucos round-trips (`⌈linhas/arraysize⌉`),
preenchendo `"_SHIFT_SEQ"`.

3. **Um** `MERGE INTO destino USING <gtt dedup> …` (§2). Roda 100% server-side.

4. `COMMIT` — o commit **zera** as linhas da GTT (limpeza automática).

5. Lê a `<err_table>` do `LOG ERRORS` (§5) para reconstruir as linhas rejeitadas
→ alimenta o branch `on_error`.

## 4. Dedup do conjunto de origem (evitar ORA-30926)

`ORA-30926` ("unable to get a stable set of rows in the source tables") dispara
quando o **source** (staging) tem **mais de uma linha** casando com a mesma linha
do destino pelas `merge_keys` — ou seja, **chaves duplicadas na origem**. O
caminho atual nunca vê isso (cada MERGE tem source de 1 linha); o set-based **vê**.

**Estratégia — dedup no próprio MERGE, no destino, "última linha vence":**

- A projeção do `USING` aplica
`ROW_NUMBER() OVER (PARTITION BY <merge_keys> ORDER BY "_SHIFT_SEQ" DESC)` e
mantém só `rn = 1` (a linha de maior `_SHIFT_SEQ`, i.e. a que chegou por último).
Isso **replica a semântica atual** (a última gravação de uma chave prevalece).

- **Métrica preservada:** `duplicates_removed = COUNT(staging) − COUNT(dedup)`,
com amostra das chaves colididas — mesmos campos que o `LoadResult` já expõe
hoje (`duplicates_removed`, `duplicate_sample`, `unique_columns`).

- O dedup app-level por `unique_columns` que o `load_service.insert` já faz
**continua** (barato, cobre a frente Python antes de tocar o banco); o dedup
DB-side por `merge_keys` é a **garantia** para todas as frentes e é o que blinda
contra a ORA-30926, independentemente de `unique_columns` ter sido configurado.

## 5. Erro por linha (LOG ERRORS / dead-letter)

Um MERGE set-based é **tudo-ou-nada** por padrão: uma linha ruim (constraint,
tipo, `NOT NULL`) aborta o statement inteiro — perdendo o diagnóstico por linha
que hoje alimenta o branch `on_error` (via `RejectedRow`).

**Estratégia — DML error logging do Oracle:**

- Anexa `LOG ERRORS INTO <err_table> ('shift') REJECT LIMIT UNLIMITED` ao MERGE.
As linhas ofensoras vão para a `<err_table>` em vez de derrubar o statement.

- A `<err_table>` é criada uma vez via
`DBMS_ERRLOG.CREATE_ERROR_LOG('<destino>', '<err_table>')` (mesmo grau de
privilégio do staging).

- Após o MERGE, `SELECT ORA_ERR_MESG$, ORA_ERR_NUMBER$, <colunas> FROM <err_table>`
reconstrói os `RejectedRow` → branch `on_error`, **preservando o formato atual**
(dead-letter: linha do cliente intacta + a coluna de controle `_shift_rejection`,
com `error`, `row_number` etc. — CIAG-595).

**Limitações conhecidas (documentadas para T1/T2):**

- `LOG ERRORS` **não** captura todos os erros — notadamente violação da constraint
única usada no próprio `ON` do MERGE e alguns erros de constraints deferidas.

- Se a `<err_table>` não puder ser criada (privilégio) **ou** o fluxo exigir
diagnóstico por linha garantido, o caminho é o **fallback row-by-row** (§6),
que continua sendo o modo de diagnóstico total. Os dois alimentam o **mesmo**
formato de `on_error`.

## 6. Flag de rollout e fallback

**Flag:** `UPSERT_STAGING_MERGE_ENABLED: bool = False` em
`app/core/config.py` (mesma convenção `_ENABLED` dos demais; **default OFF**).
Enquanto OFF, o comportamento é **exatamente** o de hoje em todas as frentes.
Na borda, o mesmo gate viaja no manifesto do job (o backend só marca
`use_staging_merge=true` quando a flag está ON e o destino é elegível), então o
aplicador Go não muda de comportamento sem o backend mandar.

**Fallback para o caminho atual (row-by-row / array-bind MERGE) quando:**

1. a flag está **OFF** (default); **ou**

2. o destino **não é Oracle** — Fase 1 é **só Oracle** (igual à épica da borda).
PG/MySQL/MSSQL seguem no upsert nativo atual; o ganho set-based deles é uma
melhoria futura mais barata (INSERT multi-linha + `ON CONFLICT`/`ON DUPLICATE`);

3. **sem privilégio de staging** — nem `CREATE TABLE` (GTT) nem
`CREATE PRIVATE TEMPORARY TABLE` em 18c+ (PTT). Detectado tentando criar e
capturando `ORA-01031` (ou versão < 18c para PTT);

4. o nó pede `returning_columns` (MERGE não tem RETURNING portável — já cai no
`_upsert_with_returning` hoje);

5. a `<err_table>` do `LOG ERRORS` não pôde ser criada **e** o fluxo exige
diagnóstico por linha.

O fallback **nunca piora**: é o comportamento de hoje. A decisão é tomada uma vez
por carga, com o motivo carimbado no resultado (para observabilidade), espelhando
o padrão de `delivery`/`fallback_reason` que o `bulk_insert` já grava para
borda-vs-túnel (CIAG-260).

## 7. Baseline medido (3 frentes)

Dataset de referência: **200.000 linhas** na `VIASOFTMCP.SPIKE_CUSTOMERS`
(8 colunas, tipos mistos — NUMBER, VARCHAR2 acentuado, DATE, TIMESTAMP, NULLs),
a mesma massa do spike CIAG-254. Oracle de teste local (VIASOFT3,
`NLS_CHARACTERSET=WE8MSWIN1252`).

| Frente | Caminho atual | Linhas | Tempo | Linhas/s | Fonte da medida |
| --- | --- | --- | --- | --- | --- |
| **Borda** (array-bind MERGE, batch 5000) | Go `oracleApplier` | 200.000 | **6,11 s** | **~32.762** | Spike CIAG-254, medido 03/jul/2026 (LAN, RTT ~0 ms) |
| **Borda** (E2E pela plataforma) | Go, via job real | 200.000 | **6,4 s** | **~31.000** | E2E 06/jul/2026 (memória `relay-borda-e2e-fixes`) |
| **Túnel** (row-by-row MERGE via WAN) | Python `_upsert_via_sqlalchemy` | 200.000 | — | **~470** (≈66× mais lento que a borda) | E2E 06/jul (relação borda/túnel); número absoluto **a confirmar** com run limpo |
| **Direta** (row-by-row) | Python `load_service` | 211.768 | ~4m17s | **~820** (append `INSERT PESSOADOC`) | Estresse CARGA CLIENTES; **upsert é ≥ isto de lento** (MERGE > INSERT). Célula upsert-específica **a preencher** |

**Como reproduzir** (harness em
`shift-backend/scripts/benchmarks/bench_upsert_baseline.py` para as frentes Python
direta/túnel, e o subcomando `apply` do spike para a borda):

```
# Direta (aponta BENCH_ORACLE_URL para o Oracle na LAN)
cd shift-backend
BENCH_ORACLE_URL='oracle+oracledb://user:pass@host:1521/?service_name=SVC' \
  python scripts/benchmarks/bench_upsert_baseline.py --rows 200000 --batch 1000

# Túnel: MESMO comando, com BENCH_ORACLE_URL apontando para host:porta do túnel do relay
# Borda: cmd/spike-oracle-apply  apply -parquet clientes_200k.parquet -batch 5000 -dsn …
```

> **Nota honesta:** as células "Borda" são **medidas** (spike + E2E, mesma massa,
> mesmo banco). As células **Direta/Túnel para o modo `upsert`** ainda não têm run
> limpo apples-to-apples porque **nenhum Oracle está acessível deste ambiente**
> (o Oracle de dev está fora do ar). Os números transcritos (~820/s append,
> ~66× borda-vs-túnel) delimitam a ordem de grandeza; o harness acima preenche as
> lacunas assim que houver um Oracle na LAN. O gate desta tarefa é ter o **método
>
>
>
> - os pontos já medidos**; T4 (CIAG-267) fecha a comparação pós-implementação.

## 8. Resumo do desenho (o que T1/T2 herdam)

- **Staging:** GTT reusada por formato (primário) · PTT 18c+ (upgrade sem
`CREATE TABLE`) · coleção descartada.

- **MERGE:** parametrizar o `USING` dos geradores existentes (`buildMergeSQL` /
`_build_merge_sql_for_dialect`) para apontar para o staging dedup; resto igual.

- **Dedup:** `ROW_NUMBER() PARTITION BY merge_keys ORDER BY _SHIFT_SEQ DESC`,
"última vence", com contagem de removidas.

- **Erro por linha:** `LOG ERRORS` + reconstrução do dead-letter; fallback
row-by-row garante diagnóstico total.

- **Flag:** `UPSERT_STAGING_MERGE_ENABLED` (default OFF); fallback nunca piora.

- **Escopo Fase 1:** só Oracle; direta+túnel = T1 (`load_service`), borda = T2 (Go).

- **UI (T3):** avisar quando as `merge_keys` não têm índice único no destino
(MERGE lento sem índice) + nota do privilégio de staging.

## 9. Rollout & benchmark (T4 / CIAG-267)

### Decisão de rollout: **manter OPT-IN** (flag OFF por padrão)

`UPSERT_STAGING_MERGE_ENABLED` **permanece `False`** por padrão. Liga-se por
ambiente/tenant quando o destino Oracle atende os pré-requisitos (§10). Motivos:

1. **O ganho ainda não foi medido em Oracle real, apples-to-apples, para o modo
upsert nas frentes direta/túnel.** O baseline (§7) tem a borda medida (spike
CIAG-254: 32k linhas/s) e a ordem de grandeza do gargalo atual (~470–820
linhas/s), mas o número do **novo** caminho por staging nessas frentes só sai
com um Oracle na LAN (o de dev está fora do ar; o CI não alcança Oracle).
Ligar por padrão antes de comprovar o ganho é ligar às cegas.

2. **O fallback precisa de rodagem em campo.** A correção do fallback (sem
privilégio de `CREATE TABLE` → caminho row-by-row; erro no MERGE → row-by-row
na frente Python; `errStagingUnavailable` na borda) está coberta por testes,
mas a variedade de privilégios dos Oracles de cliente pede validação real
antes de tornar o caminho novo o default.

3. **Custo de manter OFF é zero:** com a flag desligada o comportamento é
**idêntico** ao de hoje em todas as frentes — sem risco de regressão.

Quando um Oracle de teste estiver disponível, rodar o benchmark (abaixo); se o
ganho se confirmar e o fallback se mostrar sólido em ≥1 cliente piloto, reavaliar
o flip para ON por padrão (não requer nova issue — é virar a flag).

### Como medir o ganho (mesmo dataset do baseline, 200k linhas)

```
# Direta / túnel (Python) — com a flag OFF (baseline) e depois ON (novo caminho):
cd shift-backend
UPSERT_STAGING_MERGE_ENABLED=false BENCH_ORACLE_URL='oracle+oracledb://…' \
  python scripts/benchmarks/bench_upsert_baseline.py --rows 200000 --twice
UPSERT_STAGING_MERGE_ENABLED=true  BENCH_ORACLE_URL='oracle+oracledb://…' \
  python scripts/benchmarks/bench_upsert_baseline.py --rows 200000 --twice
# (o mesmo comando com BENCH_ORACLE_URL apontando ao host:porta do túnel = frente túnel)

# Borda (Go): via plataforma, com delivery_mode=edge e a flag ON no backend
# (o manifesto passa a carregar use_staging_merge=true). Comparar rows_per_sec
# do job com o baseline da borda.
```

Registrar aqui os `linhas/s` de cada frente (OFF vs ON) quando medidos.

### Ajustes finos (tuning) — o que já está e o que é opcional

- **Mecanismo:** GTT reusada por formato (§3), criada 1×. Sem staging permanente.

- **Lote de carga da staging:** na frente Python o `bulk_insert` já entrega o
dataset ao `load_service` **em chunks de `batch_size`** (default 1000). Como o
MERGE roda por chamada, cada chunk vira **um** MERGE set-based (ganho grande vs
1-por-linha), mas não é um único MERGE de todo o dataset. **Aumentar
`batch_size` no nó** amplia cada MERGE (menos statements) — a alavanca de tuning
principal. Na borda o array binding carrega a GTT em lotes de 5000 (sweet spot
do spike) e o MERGE é **um só** para o Parquet inteiro.

- **`ENABLE PARALLEL DML`:** *não* habilitado por padrão (exige privilégio e nem
todo destino se beneficia). É a próxima alavanca se o MERGE do destino for o
gargalo — avaliar por cliente, não ligar cego.

- **Limpeza:** automática — `ON COMMIT DELETE ROWS` zera a GTT no commit final.

### Sem regressão de correção

A idempotência é do desenho (MERGE por chave + dedup "última vence") e está
coberta por testes automatizados: frente Python
(`tests/test_upsert_staging_merge_ciag264.py` — inclui integração gated por
`BENCH_ORACLE_URL` com insere/atualiza/no-op/roda-2×) e borda
(`shift-connector/internal/apply/apply_test.go` — geração do MERGE/dedup). Com a
flag OFF (default) o caminho é bit-a-bit o atual.

## 10. Nota de operação

Para **ligar** o upsert por staging num ambiente/tenant (`UPSERT_STAGING_MERGE_ENABLED=true`):

- **Privilégio no usuário do banco de destino (direta/túnel) e no usuário de
aplicação do relay (borda):** `CREATE TABLE` (para a GTT). Sem ele, a carga
**não falha** — cai automaticamente no caminho atual (row-by-row / MERGE por
lote), só sem o ganho. Em Oracle 18c+, `CREATE PRIVATE TEMPORARY TABLE` cobre o
caso sem `CREATE TABLE` (mecanismo PTT do §3, quando implementado).

- **Índice nas `merge_keys`:** o ganho real depende de índice único/PK nas chaves
no destino (senão o próprio MERGE varre a tabela). A UI já avisa (T3/CIAG-266);
a recomendação é criar o índice via DBA.

- **Fallback observável:** quando o staging não é possível, o comportamento
degrada para o caminho atual silenciosamente (log em nível INFO/WARNING no
backend; a borda registra `errStagingUnavailable` e segue). Nada quebra.

- **Escopo Fase 1:** só **Oracle**. PG/MySQL/MSSQL seguem no upsert nativo atual
independentemente da flag.
