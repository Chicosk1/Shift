---
title: HTTP Request
---

**Categoria:** Entradas
**Tipo interno:** `http_request`

## Descrição

Faz uma requisição HTTP para qualquer endpoint e disponibiliza a resposta como dataset. É o nó de integração genérico: conecta o Shift a APIs REST, endpoints internos, webhooks de saída e qualquer serviço acessível via HTTP.

A resposta é automaticamente parseada como JSON quando possível; caso contrário, é tratada como texto. O resultado fica materializado em DuckDB para consumo pelos nós seguintes.

## Valores dinâmicos

Todos os campos onde faz sentido — **URL, query parameters, headers, autenticação e body** — aceitam valores dinâmicos, não apenas literais fixos. Cada campo oferece o seletor **fixo ↔ dinâmico** (o mesmo componente usado em Mapper, SQL Script etc.), permitindo:

- Referenciar saída de nós anteriores: `{{node_x.campo}}` (arraste o campo do painel esquerdo ou digite).

- Referenciar variáveis do fluxo: `{{vars.NOME}}`.

- Aplicar **transformações** ao valor (maiúsculo, trim, padrão/fallback, de-para etc.).

A chave de cada header/query param permanece literal; apenas o **valor** é dinâmico. Configurações já salvas no formato antigo continuam funcionando — ao editar, são migradas automaticamente para o formato dinâmico.

## Saída produzida

```
{
  "status": "completed",
  "output_field": "data",
  "status_code": 200,
  "response_headers": { "content-type": "application/json", "..." },
  "data": { "storage_type": "duckdb", "..." }
}
```

Além do dataset em `data` (ou o campo configurado), a saída sempre inclui `status_code` e `response_headers`, úteis para diagnóstico e para lógica condicional em nós seguintes.

## Configurações

### Aba Parameters

| Campo | Tipo | Padrão | Descrição |
| --- | --- | --- | --- |
| `method` | enum | `GET` | Verbo HTTP: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS` |
| `url` | valor dinâmico | — | **Obrigatório.** URL do endpoint. Aceita literais e tokens `{{node_x.campo}}` / `{{vars.X}}` |
| `auth` | objeto | `{type: "none"}` | Autenticação (ver abaixo) |
| `query_params` | lista chave→valor | `[]` | Parâmetros de query string. Chave literal, valor dinâmico |
| `headers` | lista chave→valor | `[]` | Headers HTTP adicionais. Chave literal, valor dinâmico |
| `body_mode` | enum | `none` | `none`, `structured` (construtor chave→valor) ou `raw` (editor JSON/texto) |
| `body_fields` | lista chave→valor | `[]` | Campos do body no modo `structured`. Token único preserva o tipo (número, booleano, objeto) |
| `body_raw` | string | `""` | Corpo cru no modo `raw`. Aceita tokens `{{...}}` embutidos |
| `body_format` | enum | `json` | No modo `raw`: `json` (parseia/serializa) ou `text` (envia string literal) |

### Aba Settings

| Campo | Tipo | Padrão | Descrição |
| --- | --- | --- | --- |
| `timeout_seconds` | float | `30.0` | Tempo máximo de espera pela resposta |
| `fail_on_error` | boolean | `true` | `true`: falha a execução em respostas 4xx/5xx. `false`: continua e expõe o status no resultado |
| `output_field` | string | `data` | Nome do campo na saída onde o body da resposta é gravado |
| `retry_policy` | objeto | — | Política de retentativa em caso de falha |

### Autenticação

| Tipo | Campos adicionais | Como funciona |
| --- | --- | --- |
| `none` | — | Sem autenticação |
| `bearer` | `token` | Injeta `Authorization: Bearer <token>` |
| `basic` | `username`, `password` | Injeta `Authorization: Basic <base64>` |
| `api_key` | `header` (padrão: `X-API-Key`), `value` | Injeta header customizado com o valor secreto |

Todos os valores de autenticação suportam templates `{{vars.X}}`, o que permite armazenar segredos em variáveis de workflow em vez de hardcoded na configuração.

### Importar via curl

A UI aceita colar um comando `curl` completo no campo de URL. O nó extrai automaticamente:

- Método HTTP (`-X`)

- URL e query params

- Headers (`-H`)

- Body (`-d`, `--data-raw`)

- Autenticação Basic (`-u user:senha`)

- Promoção automática de GET → POST quando body está presente

## Comportamentos da resposta

| Situação | Comportamento |
| --- | --- |
| `Content-Type: application/json` | Body parseado como JSON |
| Outro content-type | Tenta parse JSON; se falhar, trata como texto |
| Resposta vazia (HEAD, 204) | Dataset com uma linha `{"status_code": <código>}` |
| 4xx / 5xx com `fail_on_error: true` | Execução falha com erro |
| 4xx / 5xx com `fail_on_error: false` | Execução continua; `status` fica `"failed"` e `output_field` é `null` |
| Timeout | Sempre falha, independente de `fail_on_error` |

## Exemplos

**Buscar dados de uma API pública:**

```
method: GET
url: https://api.exemplo.com/pedidos
query_params: { status: "pendente", limit: "100" }
auth: { type: "bearer", token: "{{vars.ApiToken}}" }
```

**Enviar dados para um endpoint (body estruturado):**

```
method: POST
url: https://api.exemplo.com/webhook
headers: { Content-Type: "application/json" }
body_mode: structured
body_fields:
  evento → "novo_registro"            (fixo)
  id     → {{abc123.id}}              (dinâmico — preserva o tipo)
```

**Body cru com tokens embutidos (JSON aninhado):**

```
method: POST
url: https://api.exemplo.com/{{vars.tenant}}/eventos
body_mode: raw
body_format: json
body_raw: |
  {
    "evento": "novo_registro",
    "payload": { "id": "{{abc123.id}}", "origem": "shift" }
  }
```

## Limites e guardrails

- `url` vazio → erro antes da execução.

- Timeout padrão de 30 s. Para endpoints lentos, aumente `timeout_seconds`.

- Headers e query_params têm todos os valores convertidos para string antes do envio.

- `body_format: text` com body não-string → o valor é convertido para string.

- `body` em requisições GET/HEAD é tecnicamente aceito mas a maioria dos servidores ignora.

## Política de retentativa

Mesmos campos dos nós CSV e Excel (`max_attempts`, `backoff_strategy`, `backoff_seconds`, `retry_on`). Especialmente útil para APIs com rate limit ou instabilidade transitória.

## Observabilidade

A saída inclui `status_code` e `response_headers`, visíveis no painel de execução para diagnóstico de falhas de integração.
