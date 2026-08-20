---
title: Extração na Borda — Design
---

> Épica **Extração na borda** (CIAG-268..273). Este documento cobre a **T1**
> (CIAG-268): o desenho do job de extração, a matriz de roteamento do nó de
> leitura, o limiar/flag de elegibilidade e o fallback. É o **espelho, na
> LEITURA**, da épica "Relay aplicador na borda" (CIAG-254..262), que fez o mesmo
> para a ESCRITA. Onde a escrita tem seu par, ele está citado.

## Problema

Quando um fluxo lê um **volume grande** de um banco que só é alcançável pelo
Relay (rede do cliente), hoje **cada linha do resultado volta atravessando o
túnel** (reverso, sobre WebSocket/yamux) até a nuvem. Os dados sobem pela
internet do cliente no protocolo cru do banco (TNS, conversacional), sem
compressão — o túnel vira o gargalo. É o mesmo problema que a **carga** tinha na
escrita.

## Solução (control plane na nuvem, data plane na borda)

O inverso da escrita: o Relay roda o **SELECT localmente** (na LAN, rápido),
grava o resultado num **Parquet** (zstd) e o **sobe direto pro bucket** por HTTPS
(fora do túnel); o Shift **baixa da nuvem**. Menos bytes pelo mesmo uplink
(Parquet comprime 3–10×) + transporte melhor que o túnel = leitura grande muito
mais rápida. Mantém **zero porta aberta** no cliente (só saída HTTPS, como a
escrita).

Diferença estrutural em relação à escrita: na escrita o Core **materializa** o
Parquet na nuvem e o relay **baixa** (presigned GET). Na leitura é o oposto — o
relay **produz** o Parquet e o **sobe** (presigned **PUT**). Por isso o
`artifact` do manifesto de extração carrega um **destino de upload**, não uma URL
de download pronta.

## Escopo da T1 (esta tarefa)

Fundação, **inerte** até as tarefas seguintes existirem (mesmo padrão da fatia da
CIAG-260, cujo despacho só nasceu depois):

- **Decisão de roteamento pura** — `app/services/relay_extract_router.py::decide`.

- **Contrato do job de extração** — `app/services/relay_extract_manifest.py::build_extract_manifest`.

- **Flag + limiar** — `RELAY_EXTRACT_ENABLED` (OFF) e `RELAY_EXTRACT_MIN_ROWS` em `config.py`.

- **UI mínima** — seletor `read_delivery_mode` no nó SELECT quando a conexão é via relay Oracle.

**Fora de escopo** (tarefas seguintes): execução do SELECT no relay (T2/CIAG-269),
upload do Parquet (T3/CIAG-270), a **âncora** de despacho/await + emissão do
presigned PUT (T4/CIAG-271), consumo no Shift (T5/CIAG-272), higiene/segurança
(T6/CIAG-273).

## Matriz de elegibilidade (roteamento)

Avaliada pela camada async de orquestração (na T4), que resolve os fatos e chama
`decide()`. Espelha a `decide()` da escrita (CIAG-260). Ordem — mais barato/
definitivo primeiro:

| # | Critério | Fonte do fato | Inelegível → |
| --- | --- | --- | --- |
| 1 | conexão é via relay | `connections.relay_id` preenchido | túnel |
| 2 | origem é Oracle | `connection_type == "oracle"` (Fase 1) | túnel |
| 3 | relay anuncia `extract:oracle` | `tunnel_relays.capabilities` (CIAG-256) | túnel |
| 4 | relay ONLINE | `last_seen_at` dentro de `RELAY_ONLINE_WINDOW_SECONDS` (90s) | túnel |
| 5 | volume estimado ≥ `RELAY_EXTRACT_MIN_ROWS` | estimativa prévia (ver abaixo) | túnel |

Capability: `capability_for("oracle") == "extract:oracle"` (formato
`extract:<tipo>`, granular como na escrita `apply:<tipo>`).

### Override `read_delivery_mode`

Campo no config do nó (`auto` | `tunnel` | `edge`):

- **`auto`** (default) → borda se elegível, senão **túnel** com o motivo.

- **`tunnel`** → sempre túnel (nem avalia a matriz).

- **`edge`** → força a borda; se **inelegível**, **erro claro**
(`RelayExtractForcedButIneligible`) — sem fallback silencioso quando o usuário
pediu a borda explicitamente.

### Fallback

Em `auto`, qualquer inelegibilidade (ou falha runtime na borda) cai pro **túnel**.
Diferença crucial em relação à escrita: **LER não tem efeito colateral**, então o
fallback é **sempre seguro** — nenhuma linha foi aplicada em lugar nenhum, não há
risco de duplicação. (Na escrita, "FAILED com lote já aplicado" era falha seca
para não duplicar; na leitura esse cuidado não existe.) `edge` explícito continua
sendo erro, não fallback.

## Estimativa de volume ANTES de executar (ponto delicado)

O critério 5 precisa do volume **antes** de rodar a query — e é aí que mora o
custo. Opções, em ordem de preferência:

1. **Heurística por estatística da tabela (preferida, sem round-trip pela rede
do cliente):** para uma query de tabela única sem filtro seletivo, usar
`ALL_TABLES.NUM_ROWS` / `USER_TABLES.NUM_ROWS` do dicionário do Oracle (barato,
estatística do otimizador). O relay já está na LAN e pode consultar isso ao
receber o job; ou o Core estima quando conhece a tabela.

2. **`COUNT(*)` prévio:** exato, mas custa **um round-trip pelo túnel** — o mesmo
gargalo que estamos tentando evitar. Só compensa quando barato (tabela pequena
já cairia no túnel de qualquer jeito) ou quando a query tem agregação/filtro
que a estatística não captura.

3. **Desconhecido (`row_count=None`):** quando não dá para estimar de forma
barata, **não desqualificamos pelo volume** — os critérios 1–4 ainda decidem, e
a leitura pode ir pra borda mesmo sem saber o tamanho. A borda é o caminho
otimizado; escolhê-la "no escuro" nunca é pior que o túnel para leitura (sem
efeito colateral). O limiar existe para **não pagar o overhead de setup**
(materializar Parquet + upload + download) em resultados pequenos.

A **decisão de qual estratégia** (heurística vs COUNT) fica na **T4** (o despacho),
que tem o contexto async (Connection, relay) para medir. `decide()` só recebe o
número (ou `None`) e aplica a regra.

## Contrato do job de extração (manifesto)

Payload do `relay_jobs.manifest` (ledger da CIAG-257), `job_type=extract_parquet`.
Compartilhável **Python↔Go**. Montado por `build_extract_manifest` (puro).
**Nunca** carrega credenciais — o relay resolve a **origem por alias** (o `name`
da conexão, replicado no `relay.yaml` do cliente; mesma decisão de alias da
escrita).

```
{
  "manifest_version": 1,
  "dialect": "oracle",          // Fase 1 só Oracle; campo p/ extensibilidade
  "job_type": "extract_parquet",
  "extract": {
    "query": "SELECT ... FROM ...",  // só leitura (SELECT/WITH), validado
    "format": "parquet",
    "compression": "zstd",
    "max_rows": 100000               // opcional (espelha max_rows do nó); null = sem limite
  },
  "artifact": {
    "key": "relay_jobs/<job_id>/data.parquet",  // destino no bucket
    "upload_url": "<presigned PUT>",            // emitido na T4; null no backend local
    "format": "parquet",
    "compression": "zstd"
  },
  "source": {
    "connection_alias": "<nome-da-conexao>"     // relay resolve credencial local
  }
}
```

Validações do builder: `query` é de leitura (`SELECT`/`WITH`); `source_alias`
presente; `artifact.key` presente; `max_rows ≥ 0` quando informado.

Campos que a T4/T2 preenchem depois (não são da T1): a URL de PUT (`artifact.upload_url`,
presigned emitido pelo Core), e o relay complementa no `report` o `sha256`/`bytes`/
`row_count` do Parquet que produziu (para o Shift verificar antes de consumir na T5).

## Flag + limiar

- **`RELAY_EXTRACT_ENABLED: bool = False`** — rollout. Default **OFF**: enquanto
`False`, TODA leitura vai pelo túnel (comportamento de hoje) e o roteamento nem
é avaliado. Espelha `RELAY_APPLY_*` na escrita.

- **`RELAY_EXTRACT_MIN_ROWS: int = 50_000`** — limiar do critério 5. Maior que o
da escrita (`RELAY_APPLY_MIN_ROWS = 10_000`) porque a leitura tem **mais etapas
de I/O** (relay grava Parquet + sobe; Shift baixa) — o setup só compensa em
volume maior. Ajustável por cliente conforme a latência da rede.

## Registro do caminho no `output_summary`

O caminho escolhido (`delivery: edge|tunnel` + motivo) é carimbado no
`output_summary` do nó, simétrico ao que o `bulk_insert` faz na escrita. Como os
nós de leitura **ainda não produzem `output_summary`** (diferente do `bulk_insert`)
e a decisão só roda com os fatos de relay que a camada async resolve, a T1 entrega
o **mecanismo pronto e testado** — `relay_extract_router.delivery_summary(decision)`
→ `{delivery, delivery_reason, relay_job_id?}` — que o despacho da **T4** mescla no
resultado do nó quando passar a rodar `decide()`. Hoje é **inerte** (flag OFF →
sempre túnel).

## Testes

- `tests/test_relay_extract_router_ciag268.py` — cada linha da matriz (direta, sem
capability, offline, abaixo do limiar, volume desconhecido) + os overrides
(`tunnel`/`edge`, inelegível força erro) + `delivery_summary`.

- `tests/test_relay_extract_manifest_ciag268.py` — contrato válido, rejeição de
não-SELECT, alias/`key` obrigatórios, `max_rows` negativo, backend local sem
presigned, e a garantia de que **nenhuma credencial** vaza no manifesto.

## Sequência da épica

T1 (esta) → (T2 CIAG-269 execução no relay, T3 CIAG-270 upload — paralelas) →
**T4 CIAG-271 (âncora: despacho/await + presigned PUT)** → T5 CIAG-272 (consumo no
Shift) → T6 CIAG-273 (higiene/segurança: delete do artefato, allowlist de LEITURA
por relay, `user_id` no job). Reaproveita ~70% da infra da borda (Oracle no relay,
bucket OCI, ledger `relay_jobs`, presigned, jobs/report). Fica **inerte até a T4**.
