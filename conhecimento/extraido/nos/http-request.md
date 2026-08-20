# Nó `http_request` — Requisição HTTP

| | |
|---|---|
| **Tipo (MCP)** | `http_request` |
| **Rótulo na interface** | HTTP Request `[CONFIRMADO-DOC]` — `guias-de-uso/nos/http-request.md § title` / "Requisição HTTP" no MCP `[CONFIRMADO-MCP]` |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — |

> ⚠️ **Atenção ao risco declarado.** `[CONFIRMADO-MCP]` O MCP classifica este nó como `read_only`,
> mas `[CONFIRMADO-DOC]` o guia diz que ele serve para *"enviar dados para um endpoint"* e o
> `method` aceita `POST`, `PUT`, `PATCH`, `DELETE`. **O rótulo `read_only` descreve o efeito sobre o
> fluxo, não sobre o mundo externo** — um `DELETE` daqui apaga dados de verdade no sistema de
> destino. — `describe_node('http_request')` vs. `guias-de-uso/nos/http-request.md § Descrição`

## O que faz

`[CONFIRMADO-MCP]` Executa uma chamada HTTP para consumir APIs REST. Suporta autenticação Bearer,
Basic e API Key, além de templates de variáveis no URL/headers/body.
— `describe_node('http_request')`

`[CONFIRMADO-DOC]` É **o nó de integração genérico**: conecta o Shift a APIs REST, endpoints
internos, webhooks de saída e qualquer serviço acessível via HTTP. A resposta é automaticamente
parseada como JSON quando possível; caso contrário, é tratada como texto. O resultado fica
materializado em DuckDB. — `guias-de-uso/nos/http-request.md § Descrição`

## Quando usar

`[CONFIRMADO-MCP]` — `describe_node('http_request')`
- Buscar dados de uma API REST externa.
- Enviar dados para um webhook ou API de terceiros.
- **Integrar com serviços que não têm conector nativo no Shift.** ← este é o caminho para
  notificação, integração com ERP e qualquer coisa fora dos 63 nós do catálogo.

## Parâmetros de configuração

### Aba *Parameters*

`[CONFIRMADO-DOC]` — `guias-de-uso/nos/http-request.md § Aba Parameters`
`[CONFIRMADO-MCP]` — `describe_node('http_request')` (marcado onde as fontes divergem)

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `method` | enum | **sim** (MCP) | `GET` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD` — e `OPTIONS` só no doc (ver divergência) |
| `url` | valor dinâmico | **sim** | — | URL do endpoint. Aceita literais e tokens |
| `auth` | objeto | não | `{type: "none"}` | Autenticação (ver abaixo) |
| `query_params` | lista chave→valor | não | `[]` | Chave literal, **valor dinâmico** |
| `headers` | lista chave→valor | não | `[]` | Chave literal, **valor dinâmico** |
| `body_mode` | enum | não | `none` | `none`, `structured`, `raw` — e `json_value` só no MCP |
| `body_fields` | lista chave→valor | não | `[]` | Modo `structured`. Token único **preserva o tipo** (número, booleano, objeto) |
| `body_raw` | string | não | `""` | Modo `raw`. Aceita tokens `{{...}}` embutidos |
| `body_format` | enum | não | `json` | Modo `raw`: `json` (parseia/serializa) ou `text` (string literal) |
| `body_value` | objeto | não | — | **Só no MCP.** Usado com `body_mode: json_value` |

> ⚠️ **DIVERGÊNCIA — superfície de configuração.** As duas fontes descrevem conjuntos de campos
> diferentes e nenhuma foi eleita:
> - `[CONFIRMADO-DOC]` `method` inclui **`OPTIONS`**; `body_mode` tem 3 opções (`none`,
>   `structured`, `raw`) e existem `body_fields`, `body_raw`, `body_format`.
>   — `guias-de-uso/nos/http-request.md § Aba Parameters`
> - `[CONFIRMADO-MCP]` `method` **não tem `OPTIONS`**; `body_mode` tem 4 opções, incluindo
>   **`json_value`** ("envia o valor de uma coluna STRUCT/LIST como corpo raiz — combina com o nó
>   Aninhar"), com o campo `body_value`. Não menciona `body_fields`/`body_raw`/`body_format`.
>   — `describe_node('http_request')`
>
> `[INFERIDO]` A leitura mais provável é que o MCP esteja mais novo (o nó `nest`/Aninhar declara
> "saída pronta para o body de uma Requisição HTTP", casando com `json_value`) e que o guia esteja
> defasado, **mas o MCP também omite campos que o guia detalha** — então não é substituição limpa.
> **Falta descobrir** o esquema real e único do nó.

### Aba *Settings*

`[CONFIRMADO-DOC]` — `guias-de-uso/nos/http-request.md § Aba Settings`

| Parâmetro | Tipo | Padrão | Descrição |
|---|---|---|---|
| `timeout_seconds` | float | **`30.0`** | Tempo máximo de espera pela resposta |
| `fail_on_error` | boolean | **`true`** | `true`: falha a execução em 4xx/5xx. `false`: continua e expõe o status |
| `output_field` | string | `data` | Campo de saída onde o body é gravado |
| `retry_policy` | objeto | — | Retentativa (só no doc; o MCP não lista) |

`[CONFIRMADO-MCP]` `timeout_seconds` (padrão 30) e `fail_on_error` (padrão `true`) confirmados
também pelo contrato do MCP. — `describe_node('http_request')`

### Autenticação

`[CONFIRMADO-DOC]` **4 tipos** — `guias-de-uso/nos/http-request.md § Autenticação`

| Tipo | Campos | Como funciona |
|---|---|---|
| `none` | — | Sem autenticação |
| `bearer` | `token` | Injeta `Authorization: Bearer <token>` |
| `basic` | `username`, `password` | Injeta `Authorization: Basic <base64>` |
| `api_key` | `header` (padrão `X-API-Key`), `value` | Header customizado com o valor secreto |

> ⚠️ **DIVERGÊNCIA — existe um 5º tipo só no MCP.** `[CONFIRMADO-MCP]`
> **`api_token_credential`** com `{credential_id}`: autentica *"por uma credencial de token de API
> cadastrada (**login→token com cache/relogin automático**)"*. Por padrão injeta
> `Authorization: Bearer <token>`; aceita `{header}` (default `Authorization`) e `{prefix}` (default
> `Bearer`) — **`prefix: ''` manda o token CRU, sem prefixo**.
> — `describe_node('http_request')`. O guia
> (`guias-de-uso/nos/http-request.md § Autenticação`) **não menciona este tipo**. É o único
> mecanismo documentado para API que exige login e devolve token temporário — caso típico de ERP.

`[CONFIRMADO-DOC]` Todos os valores de autenticação suportam templates `{{vars.X}}`, o que permite
**armazenar segredos em variáveis de workflow em vez de hardcoded na configuração**.
— `guias-de-uso/nos/http-request.md § Autenticação`

### Valores dinâmicos e sintaxe de token

`[CONFIRMADO-DOC]` **URL, query params, headers, autenticação e body** aceitam valores dinâmicos.
Cada campo oferece o seletor **fixo ↔ dinâmico** (mesmo componente do Mapper e do SQL Script),
permitindo referenciar saída de nós anteriores (`{{node_x.campo}}`), variáveis do fluxo
(`{{vars.NOME}}`) e aplicar **transformações** ao valor (maiúsculo, trim, padrão/fallback, de-para).
**A chave de cada header/query param permanece literal; apenas o valor é dinâmico.** Configurações
salvas no formato antigo continuam funcionando e são migradas ao editar.
— `guias-de-uso/nos/http-request.md § Valores dinâmicos`

> 🔥 `[CONFIRMADO-MCP]` **A sintaxe do token exige o id REAL do nó.**
> *"`{{node_<id>.<campo>}}` referencia um campo da saída de um nó upstream (id REAL do nó — ex.:
> `{{node_a1b2c3.cep}}`); `{{vars.X}}` referencia variável do fluxo. **NUNCA use `{{campo}}` solto
> nem `{{data.campo}}` — não resolvem em runtime.**"* — `describe_node('http_request')`
>
> Esta é a pegadinha mais fácil de cair: `{{data.campo}}` parece natural (o campo de saída padrão
> *chama* `data`) e falha silenciosamente na resolução.

### Importar via curl

`[CONFIRMADO-DOC]` A UI aceita **colar um comando `curl` completo no campo de URL**. O nó extrai
automaticamente: método (`-X`), URL e query params, headers (`-H`), body (`-d`, `--data-raw`), auth
Basic (`-u user:senha`) e faz **promoção automática de GET → POST quando há body**.
— `guias-de-uso/nos/http-request.md § Importar via curl`

## Entradas esperadas

`[INFERIDO]` Nenhuma obrigatória — está na categoria `input` e pode ser o primeiro nó depois do
gatilho. Mas aceita upstream: os tokens `{{node_<id>.<campo>}}` só resolvem se houver um nó anterior
ligado, e `body_mode: json_value` `[CONFIRMADO-MCP]` consome uma coluna STRUCT/LIST do upstream
(tipicamente vinda do nó `nest`/Aninhar). — `describe_node('http_request')` e `describe_node`/
`list_nodes` para `nest`

`[LACUNA]` **Não está documentado se o nó dispara uma requisição por linha do upstream ou uma só
por execução.** Comparação: `gmail_send` e `zapi_send_text` declaram explicitamente "uma por
execução — para uma por linha, envolva num nó Loop"; o `http_request` **não faz essa declaração em
nenhuma das duas fontes**. Falta descobrir. Para uma automação de preço que precisa chamar a API uma
vez por produto, é a diferença entre funcionar e enviar só o primeiro item.

## Saídas produzidas

`[CONFIRMADO-DOC]` Além do dataset em `data` (ou o campo configurado), a saída **sempre** inclui
`status_code` e `response_headers`, úteis para diagnóstico e para lógica condicional em nós
seguintes. — `guias-de-uso/nos/http-request.md § Saída produzida` e § Observabilidade

```json
{"status": "completed", "output_field": "data", "status_code": 200,
 "response_headers": {"content-type": "application/json"},
 "data": {"storage_type": "duckdb"}}
```

### Comportamentos da resposta

`[CONFIRMADO-DOC]` — `guias-de-uso/nos/http-request.md § Comportamentos da resposta`

| Situação | Comportamento |
|---|---|
| `Content-Type: application/json` | Body parseado como JSON |
| Outro content-type | Tenta parse JSON; se falhar, trata como texto |
| Resposta vazia (HEAD, 204) | Dataset com uma linha `{"status_code": <código>}` |
| 4xx/5xx com `fail_on_error: true` | Execução **falha** |
| 4xx/5xx com `fail_on_error: false` | Execução continua; `status` fica `"failed"` e `output_field` é `null` |
| **Timeout** | **Sempre falha, independente de `fail_on_error`** |

> `[CONFIRMADO-MCP]` Para resposta **XML**, o caminho é o nó `xml_to_rows`: *"é o nó para tratar
> resposta de API que devolve XML — o corpo de um nó HTTP não-JSON chega na coluna `value` e é
> consumido direto, sem salvar arquivo."* — `list_nodes()` (descrição de `xml_to_rows`)

## Erros comuns

`[CONFIRMADO-DOC]` — `guias-de-uso/nos/http-request.md § Limites e guardrails`
- **`url` vazio** → erro **antes** da execução.
- **Timeout padrão de 30 s.** Para endpoints lentos, aumente `timeout_seconds`.
- **Headers e `query_params` têm todos os valores convertidos para string** antes do envio.
- **`body_format: text` com body não-string** → o valor é convertido para string.
- **`body` em GET/HEAD** é tecnicamente aceito, mas a maioria dos servidores **ignora**.

`[CONFIRMADO-MCP]` `{{campo}}` solto e `{{data.campo}}` **não resolvem em runtime** — use o id real
do nó. — `describe_node('http_request')`

`[CONFIRMADO-DOC]` **Política de retentativa** especialmente útil para APIs com rate limit ou
instabilidade transitória. Mesmos campos dos nós CSV e Excel (`max_attempts`, `backoff_strategy`,
`backoff_seconds`, `retry_on`). — `guias-de-uso/nos/http-request.md § Política de retentativa`

`[LACUNA]` **`fail_on_error: false` produz um estado ambíguo**: a execução continua, o `status` do
nó fica `"failed"` e `output_field` é `null` — mas não está documentado se a **execução do fluxo**
termina como `COMPLETED` ou `FAILED`, nem se o nó seguinte recebe um dataset vazio ou nada.
Consequência direta para observabilidade (ver `procedimentos/monitorar-execucao.md`): uma API que
respondeu 500 pode virar "rodou e não achou nada".

## Exemplo

`[CONFIRMADO-MCP]` Do contrato — *"Como buscar dados de uma API com autenticação Bearer?"*:

```json
{"method": "GET", "url": "https://api.exemplo.com/dados",
 "auth": {"type": "bearer", "token": "{{vars.API_TOKEN}}"}, "output_field": "data"}
```

`[CONFIRMADO-MCP]` *"Como usar um campo de um nó anterior na URL (ex.: consultar o CEP que chega do
trigger)?"*:

```json
{"method": "GET", "url": "https://viacep.com.br/ws/{{node_a1b2c3.cep}}/json/",
 "output_field": "data"}
```

`[CONFIRMADO-MCP]` *"Como consultar uma API de ERP que exige login e devolve um token temporário?"*
— o caso do `api_token_credential`:

```json
{"method": "GET", "url": "https://erp.exemplo.com/api/v2/pedidos",
 "auth": {"type": "api_token_credential",
          "credential_id": "<uuid da credencial cadastrada em Conexões>"},
 "output_field": "data"}
```

`[CONFIRMADO-DOC]` Body cru com tokens embutidos (JSON aninhado)
— `guias-de-uso/nos/http-request.md § Exemplos`:

```
method: POST
url: https://api.exemplo.com/{{vars.tenant}}/eventos
body_mode: raw
body_format: json
body_raw: |
  {"evento": "novo_registro", "payload": {"id": "{{abc123.id}}", "origem": "shift"}}
```

`[LACUNA]` **Não há exemplo em aula.** A transcrição do lote (`modulo-3-parte-4-saidas.txt`) não
cobre o `http_request`. Falta descobrir o rótulo literal na tela das abas *Parameters* e *Settings*
(os nomes aqui vêm do documento, não de UI observada) e como o seletor fixo ↔ dinâmico aparece.
