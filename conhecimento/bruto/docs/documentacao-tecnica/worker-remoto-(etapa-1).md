---
title: Worker Remoto (Etapa 1)
---

## Ligar / desligar

**Ligar** (roteia `transform_node`, `bulk_insert`, `sql_database`, `code_node` para o worker):

```
# .env
SHIFT_WORKER_URL=http://shift-worker:8001
SHIFT_WORKER_NODE_TYPES=transform_node,bulk_insert,sql_database,code_node
SHIFT_WORKER_AUTH_TOKEN=<mesmo valor nos dois services>
```

```
docker compose up -d shift-worker
```

**Desligar** (tudo roda in-process no Core, comportamento pré-Etapa 1):

```
SHIFT_WORKER_NODE_TYPES=
```

Não é necessário parar o container — basta esvaziar a variável e reiniciar o Core.
O worker continua respondendo `/health` mas não recebe tráfego.

---

## Diagnosticar fallback

O Core caiu para execução local quando o worker estava inacessível se você ver:

**Log** (Core):

```
node.processor.worker_unreachable_fallback_local  processor_type=bulk_insert  error="Worker inacessivel: ..."
```

**Métrica**:

```
curl -s http://localhost:8000/metrics | grep shift_worker_fallback_total
```

Para ver todas as chamadas ao worker e seus desfechos:

```
curl -s http://localhost:8000/metrics | grep shift_worker_call
```

Desfechos possíveis em `outcome`: `success`, `skipped`, `processing_error`, `unreachable`, `unexpected_error`.

---

## Ajustar SHIFT_WORKER_NODE_TYPES

A variável é um CSV de `node_type`s. Qualquer subconjunto funciona:

```
# Só transform (mais leve, bom para testar a infra):
SHIFT_WORKER_NODE_TYPES=transform_node

# Tudo exceto code_node (sem precisar de docker.sock no worker):
SHIFT_WORKER_NODE_TYPES=transform_node,bulk_insert,sql_database

# Tudo (configuração padrão de produção):
SHIFT_WORKER_NODE_TYPES=transform_node,bulk_insert,sql_database,code_node
```

Requer reiniciar o **Core** para recarregar a env var. O worker não precisa reiniciar.

---

## Dimensionar o pool de threads (`SHIFT_NODE_POOL_SIZE`)

Cada processo executa nós num pool de threads dedicado (`shift-node-*`). Os dois
lêem a **mesma** variável, e em produção o compose define valores diferentes:

| Processo | Valor em prod | Racional |
| --- | --- | --- |
| Core | `24` | Absorve a carga interativa (API + SSE + agentes) além da execução local |
| Worker | `10` | Só executa o pesado; pool grande aqui rouba conexão de banco sem ganho |

O valor do Worker é **literal** no `docker-compose.prod.yml` — precisa ser
override do que o `.env` diz para o Core, junto com `SHIFT_WORKER_URL`,
`SHIFT_WORKER_NODE_TYPES` e `SHIFT_SCHEDULER_ENABLED`.

> ⚠️ **`SHIFT_THREADPOOL_MAX_WORKERS` está morta.** Era o nome do CIAG-20;
> desde o CIAG-296 nenhuma linha de código a lê. Enquanto o compose a definia,
> os dois containers rodavam no default `16` e a assimetria 24/10 acima não
> existia. Se o `.env` de produção ainda tiver a variável antiga, troque pelo
> nome novo — ela não gera erro nem aviso, só é ignorada.

Ocupação em tempo real: `curl -s http://localhost:8000/metrics | grep node_pool`
(`node_pool_busy` / `node_pool_size`). O alerta `ShiftNodePoolSaturated` dispara
com o pool 100% ocupado por 10 min.

---

## Teto de RAM (`SHIFT_MAX_EXECUTION_MEMORY_MB`)

Só o **Core** roda o monitor de memória (`app/services/memory_monitor.py`): ao
passar do teto por 3 leituras seguidas, ele encerra a execução mais antiga como
`CRASHED`. O Worker executa nós avulsos e não tem registro de execução para
eleger uma vítima.

O teto tem de ficar **abaixo** do `mem_limit` do container. Com o default do
código (`4096`) e `mem_limit: 2g`, o OOM kill do Docker chegava primeiro e
derrubava o container inteiro — levando junto todas as execuções em voo. Prod
usa `1536`.

---

## Quando NÃO usar o worker

- **Dev local single-user**: o worker é um processo a mais sem ganho real. Deixe `SHIFT_WORKER_NODE_TYPES=` vazio.

- **Máquina com pouca RAM**: cada container consome ~2 GB de limite. Com worker ativo, o mínimo sobe para ~4 GB.

- **Sem acesso a `docker.sock`**: `code_node` no worker precisa criar containers de sandbox. Sem o socket montado, ele cai no cold path; os demais nodes continuam funcionando normalmente.

- **Debug de um node específico**: esvaziar `SHIFT_WORKER_NODE_TYPES` faz o node rodar in-process com stack trace completo no terminal do Core, sem precisar olhar logs de dois containers.
