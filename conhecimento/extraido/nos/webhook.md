# Nó `webhook` — Trigger Webhook

| | |
|---|---|
| **Tipo (MCP)** | `webhook` |
| **Rótulo na interface** | Webhook |
| **Categoria** | `trigger` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — (nenhum sucessor declarado nas fontes) |

## O que faz

`[CONFIRMADO-MCP]` Inicia o workflow quando um payload HTTP é recebido na rota
`/api/v1/webhook/{path}`. Repassa `method`, `headers`, `query_params` e `body` para os nós
seguintes. **Materializa o body como dataset DuckDB quando for dict/lista de dicts.**
— `describe_node('webhook')`

`[CONFIRMADO-DOC]` Expõe uma URL HTTP **pública**. Permite integrar o Shift a qualquer sistema
externo que suporte envio de webhooks — ERPs, plataformas de e-commerce, ferramentas de
automação. — `nos/webhook-(gatilho).md § Descrição`

`[VÍDEO]` Em aula, demonstrado com o Postman: *"Ele permite que você gere um webhook dentro da
plataforma"*, e depois *"Então aqui ele gera uma URL, e você pode usar ela pra chamar alguma
coisa"*. — `m3-2A:26-34`

## Duas URLs, dois mundos

`[CONFIRMADO-DOC]` Cada nó Webhook gera **duas URLs distintas**
(`nos/webhook-(gatilho).md § Descrição`):

| URL | Rota | Quando usar |
|---|---|---|
| **Test URL** | `/api/v1/webhook-test/{path}` | Testes em desenvolvimento; o workflow roda em modo draft |
| **Production URL** | `/api/v1/webhook/{path}` | Uso real; **só com o workflow em Produção e Publicado** |

`[VÍDEO]` A aula confirma pelos dois lados, incluindo o erro:
- Teste: *"para você testar, muda aqui pra teste, vou usar o... como é teste **não precisa
  publicar**"*. — `m3-2A:35`
- Produção: *"Eu posso mudar ele pra produção aqui, obviamente, né, e publicar. Lembrando, né,
  sempre publica (...) e daí obviamente eu vou usar aqui a URL de produção"*. — `m3-2A:44`
- E o resultado do teste em produção foi **404 Not Found**, mas por causa do ambiente, não do
  nó: *"É, homologação ele é bloqueado (...) Mas eu deixei bloqueado produção por questão de não
  rodar nada aqui (...) Mas aqui em teste você consegue"*. — `m3-2A:45-47`

> `[UI-OBSERVADA]` **Fato específico do ambiente de homologação:** a URL de **produção** do
> webhook está **bloqueada em homologação** por decisão do instrutor/administrador; a de **teste**
> funciona. Isso é configuração de ambiente, não comportamento do nó — mas explica um `404` que
> pareceria bug. — `m3-2A:46-47`

## Quando usar

`[CONFIRMADO-MCP]`
- Disparar o workflow via POST de um sistema externo (ERP, formulário, API parceiro).
- Receber eventos de webhook (ex.: pagamento confirmado, pedido criado).
- Processar o body de um payload HTTP como tabela de dados.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('webhook')`

| Parâmetro | Tipo | Padrão | Descrição |
|---|---|---|---|
| `http_method` | enum | `POST` | `GET` / `POST` / `PUT` / `PATCH` / `DELETE` / `HEAD` |
| `path` | string | `workflow_id` | Path customizado da URL pública. Se omitido, usa o `workflow_id` |
| `authentication` | object | `none` | `{type: none\|bearer\|header\|basic\|jwt, ...}` — ver abaixo |
| `respond_mode` | enum | `immediately` | `immediately` / `on_finish` |
| `response_code` | number | `200` | Código HTTP de resposta |
| `response_data` | enum | `first_entry_json` | `first_entry_json` / `all_entries` / `no_body` |
| `raw_body` | boolean | `false` | Guarda o corpo cru em **base64**, sem tentar parse JSON |
| `allowed_origins` | string | *(vazio)* | CSV de origens permitidas (CORS) ou `*`. Vazio **desabilita** CORS |
| `output_field` | string | `data` | Nome do campo de saída do body |
| `validation_mode` | enum | `lenient` | `lenient` / `strict` — ver abaixo |
| `input_schema` | array | — | Contrato de entrada declarado — ver abaixo |

`[UI-OBSERVADA]` Os mesmos grupos aparecem na tela da aula: *"aqui as entradas, se você tem
algum tipo de autenticação. A resposta, se vai responder imediatamente ou ao terminar o fluxo,
como que vai ser o corpo da resposta (...) você só permite que venha dados de determinada URL ou
de determinado IP. Então você pode definir **origens permitidas** aqui também."* — `m3-2A:31`

`[UI-OBSERVADA]` O `path` é **gerado aleatório** e pode ser trocado: *"Você pode definir o path,
então ele gera sempre aqui algo aleatório, mas você poderia colocar aqui sei lá, funcionários."*
— `m3-2A:30`. `[CONFIRMADO-DOC]` A doc detalha: é um **UUID gerado automaticamente na primeira
abertura**. — `nos/webhook-(gatilho).md § Aba Parameters`

> ⚠️ **DIVERGÊNCIA menor — métodos HTTP.** `[VÍDEO]` A aula lista *"POST, PUT, PATCH, DELETE ou
> **READ**"* (`m3-2A:29`). `[CONFIRMADO-MCP]` + `[CONFIRMADO-DOC]` O contrato e a doc listam
> `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`. **Ambas registradas.** Leitura provável
> (`[INFERIDO]`): "READ" é lapso de fala para `GET` ou `HEAD`. Não muda nada de operacional, mas
> a aula **omite `GET`**, que existe.

### `input_schema` — contrato de entrada (só no MCP)

`[CONFIRMADO-MCP]` Lista de campos `{name, type, location, required, default}`:

- `type`: `string` / `integer` / `float` / `boolean` / `date` / `datetime` / `object` / `array`
- `location`: `body` / `query` / `header`
- **Toda coluna declarada existe na saída** (default/NULL se ausente) e é **tipada via
  `TRY_CAST`**.
- Campos de **query e header também entram no dataset** — header é *case-insensitive*, útil para
  `GET` e para tokens/assinaturas em header.

`[CONFIRMADO-MCP]` `validation_mode` governa a violação do schema:

| Valor | Comportamento |
|---|---|
| `lenient` | **Padrão.** Preenche default/NULL e segue |
| `strict` | Recusa com **HTTP 422**, listando o que faltou / tipo inválido |

`[UI-OBSERVADA]` A aula mostra exatamente essa declaração na tela, sem nomear o parâmetro:
*"você poderia estar adicionando dados aqui, adicionar um campo no body que você espera, um campo
na query que você espera, um campo no header que você espera (...) Você adicionaria aqui o campo,
o tipo dele."* — `m3-2A:34`

> `[LACUNA]` `input_schema` e `validation_mode` **não aparecem** em
> `nos/webhook-(gatilho).md`. A doc de uso descreve a aba Parameters sem eles. São exclusivos do
> contrato do MCP + confirmados de vista na aula. Falta descobrir se a doc está desatualizada ou
> se a UI expõe isso em outro lugar.

### Autenticação

`[CONFIRMADO-DOC]` — `nos/webhook-(gatilho).md § Autenticação`

| Tipo | Campos adicionais | Descrição |
|---|---|---|
| `none` | — | Sem autenticação — **qualquer chamada é aceita** |
| `bearer` | `bearer_token` | Exige `Authorization: Bearer <token>`. Informe **apenas o token** — o prefixo é do protocolo |
| `header` | `header_name`, `header_value` | Exige header com valor secreto (ex.: `X-Webhook-Secret`) |
| `basic` | `username`, `password` | HTTP Basic Auth |
| `jwt` | `jwt_secret`, `jwt_algorithm` | JWT; algoritmos HS256, HS384, HS512, RS256 |

#### Cofre de Segredos — o caminho recomendado

`[CONFIRMADO-MCP]` O contrato é explícito: *"**PREFIRA** apontar para o Cofre de Segredos com
`{secret_id}`"*. Os campos literais (`bearer_token`, `header_value`, `password`, `jwt_secret`)
*"seguem aceitos por retrocompatibilidade, mas **gravam o valor em claro no fluxo** — **não use em
fluxo novo**"*. — `describe_node('webhook')`

`[CONFIRMADO-MCP]` + `[CONFIRMADO-DOC]` **Modo do segredo por tipo** — as duas fontes batem:

| Tipo | Modo exigido | Por quê |
|---|---|---|
| `bearer`, `header`, `basic` | `verify` | O Shift só precisa **comparar** o que chegou com o que guardou |
| `jwt` | **`reveal`** | Conferir a assinatura HS256 exige o **valor original** |

`[CONFIRMADO-MCP]` + `[CONFIRMADO-DOC]` Um webhook `jwt` apontando para segredo `verify` é
**recusado ao publicar**, com mensagem dizendo o que fazer — *"não vira erro obscuro em
execução"*.

`[CONFIRMADO-DOC]` Ganhos operacionais do cofre (`nos/webhook-(gatilho).md § Segredo vindo do
Cofre`):
- O segredo **sai de dentro do fluxo** — quem exporta o desenho vê que existe proteção, não qual é.
- **Trocar o token não exige mexer no fluxo**; é rotação no cofre. Durante a **janela de graça** o
  token antigo ainda é aceito, então a integração não cai.
- **Revogar vale na hora, em todos os fluxos** que usam o segredo. Bom numa emergência — e vale
  saber antes de revogar um segredo de rotina, porque ele pode estar em mais de uma integração.
- Campos que **não são segredo** ficam no nó: `header_name` e, no Basic, `username`.
- **Se referência e valor literal estiverem os dois preenchidos, a referência vence** — o valor
  digitado é ignorado.
- Ao publicar, o Shift confere a referência: segredo inexistente, revogado, vencido, de outro
  Espaço ou de modo incompatível **barra a publicação**.
- Já em produção, chamada com segredo inválido recebe **`401`** e aparece na trilha de chamadas
  recebidas como falha de autenticação. A recusa é sempre **a mesma mensagem genérica, de
  propósito** — dizer "esse segredo está revogado" entregaria informação a quem tenta adivinhar.

`[CONFIRMADO-MCP]` Se você não sabe o id do segredo: deixe `secret_id` vazio e peça à pessoa para
selecionar no seletor do cofre.

### Modo de resposta (`respond_mode`)

> ⚠️ **DIVERGÊNCIA — três modos na doc, dois no contrato.** **Ambas registradas:**
>
> - `[CONFIRMADO-DOC]` `nos/webhook-(gatilho).md § Modo de resposta` lista **três**:
>   `immediately`, `on_finish` e **`using_respond_node`** — este último descrito como *"a resposta
>   é controlada por um nó 'Respond to Webhook' no fluxo"*.
> - `[CONFIRMADO-MCP]` `describe_node('webhook')` declara o enum com **dois** valores:
>   `immediately` e `on_finish`. Não há `using_respond_node`.
>
> **Agravante que reforça o lado do MCP:** o nó "Respond to Webhook" **não existe no catálogo**.
> `list_nodes(query='respond')` não retorna nenhum nó de resposta a webhook, e
> `describe_node('respond_to_webhook')` devolve *"Tipo de nó não encontrado"*.
> `[LACUNA]` Falta descobrir se o modo foi removido, se ainda não foi implementado, ou se o nó
> existe sob outro nome. **Não use `using_respond_node`** sem confirmar.

| Valor | Comportamento |
|---|---|
| `immediately` | **Padrão.** Responde `200 OK` assim que o payload é recebido, **sem esperar** o workflow terminar |
| `on_finish` | Mantém a conexão aberta e responde **somente quando o workflow conclui**, com a saída do último nó |

`[UI-OBSERVADA]` Na tela: *"A resposta, se vai responder imediatamente ou ao terminar o fluxo"*.
— `m3-2A:31`

#### ⚠️ `immediately` é at-least-once — **pode executar duas vezes**

`[CONFIRMADO-DOC]` + `[CONFIRMADO-MCP]` As duas fontes dizem o mesmo, e é o fato mais importante
deste nó:

> `[CONFIRMADO-DOC]` *"O preço dessa garantia é que o fluxo **pode rodar duas vezes**. Se a
> execução for interrompida no meio, o disparo volta para a fila e é executado de novo — o Shift
> **prefere repetir a perder**."* — `nos/webhook-(gatilho).md`

Mecanismo `[CONFIRMADO-DOC]`: no modo `immediately` a chamada é **gravada numa fila durável** e a
resposta sai na hora; quem executa o fluxo é um **processo separado**, logo em seguida. Antes o
disparo vivia só na memória e um reinício o perdia em silêncio. Agora o reinício não perde nada —
ao custo da reexecução.

`[CONFIRMADO-MCP]` O contrato repete a obrigação: *"pode reexecutar depois de uma queda, então
fluxos que gravam em sistemas externos **devem ser idempotentes**"*.

`[CONFIRMADO-DOC]` Mitigação declarada: *"use uma chave do payload para verificar se aquele
registro já foi processado antes de gravar"*. E: *"o histórico de execuções mostra quantas vezes o
fluxo realmente rodou para cada chamada"*.

`[CONFIRMADO-DOC]` **Vale só para `immediately`.** No `on_finish` o fluxo roda **inline**, porque
a resposta É o resultado dele.

## Entradas esperadas

**Nenhuma.** É gatilho — só tem saída. Ver a regra estrutural em `m3-2A:5` `[VÍDEO]`: um único nó
de gatilho por fluxo, obrigatório.

## Saídas produzidas

`[CONFIRMADO-DOC]` — `nos/webhook-(gatilho).md § Saída produzida`

```json
{
  "trigger_type": "webhook",
  "status": "triggered",
  "http_method": "POST",
  "headers": { "content-type": "application/json" },
  "query_params": { "source": "erp" },
  "data": { "...body da requisição..." }
}
```

O body fica no campo definido por `output_field` (padrão `data`); query strings em `query_params`;
cabeçalhos em `headers`.

`[CONFIRMADO-MCP]` Quando o body é dict ou lista de dicts, ele é **materializado como dataset
DuckDB** — ou seja, já vira tabela, sem passar por nó de desaninhar.

`[VÍDEO]` A aula tropeça exatamente nisso e a lição fica registrada: o instrutor liga um nó
**Desaninhar** depois do webhook e toma erro — *"deu um erro aqui. É, do tipo varchar, nem
precisava então desaninhar porque ele já, ele só veio esse dado (...) eu já podia usar direto aqui
num nó mapeamento"*. — `m3-2A:39-41`

## Erros comuns

`[CONFIRMADO-DOC]` — `nos/webhook-(gatilho).md § Limites e guardrails`

- **Production URL só fica acessível** após o workflow ser colocado em Produção **e** publicado.
- `path` vazio → o **ID do workflow** é usado como path de fallback.
- Com `respond_mode: on_finish`, chamadores com **timeout curto** podem receber erro de conexão
  encerrada antes de o workflow concluir.
- `raw_body: true` com `GET` ou `HEAD` é **bloqueado pela UI** (não carregam body).
- **Plataforma lotada + `on_finish` → `429 Too Many Requests` imediato**, com header
  `Retry-After: 30`. Quando o teto de execuções simultâneas (global **ou do projeto**) está
  atingido, a chamada é **recusada na hora, sem enfileirar**. Aparece na auditoria como
  *"Rejeitado: plataforma lotada"*, com o conteúdo recebido.
- **No `immediately` não existe esse 429**: a chamada não disputa vaga, entra na fila durável. Uma
  rajada é aceita e guardada. O trabalho de fundo tem **orçamento próprio de execução**, para que
  um pico de webhooks não tire a vaga de quem está usando a plataforma na tela.
- Se nem gravar a chamada na fila for possível (banco indisponível) → **`503`**. O Shift não
  responde "aceitei" para algo que não conseguiu registrar.
- `authentication: none` significa **qualquer chamada é aceita** numa URL pública.

`[CONFIRMADO-DOC]` **`webhook` não é exportável** para SQL/Python — está na lista de HTTP 422,
grupo *"I/O externa"*. — `guias-de-uso/exportar-e-importar.md § Cobertura V1`

## Como testar

`[CONFIRMADO-DOC]` — `nos/webhook-(gatilho).md § Como testar`

1. Abra o nó no editor e clique em **Listen for test event**.
2. O Shift fica aguardando por até **120 segundos**.
3. Dispare uma requisição para a **Test URL** exibida no painel.
4. O payload capturado é **injetado automaticamente** na execução de teste.

`[VÍDEO]` É exatamente o que a aula faz, e o rótulo de resultado na tela é **"capturado"**:
*"eu posso colocar aqui pra escutar, né, esperar. E quando eu rodar, ele vai dar o status
capturado"*. O corpo enviado pelo Postman foi `{"msg": "hello"}`. Depois o instrutor repete o
ciclo com um nó de Mapeamento aplicando maiúsculo: *"Vou botar executar para ele escutar, enviar
de volta. Agora sim, já ficou maiúsculo."* — `m3-2A:37`, `m3-2A:43`

> `[UI-OBSERVADA]` O ciclo de teste é **manual e sincronizado**: você clica em executar/escutar
> **primeiro** e disparo externo **depois**, dentro da janela de 120 s. Não é um listener
> permanente.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
// Receber um POST protegido por token, sem deixar o token salvo dentro do fluxo
{"http_method": "POST", "authentication": {"type": "bearer", "secret_id": ""}}

// POST de um ERP com cnpj obrigatório (texto) e uf opcional, recusando quem não mandar o cnpj
{"http_method": "POST", "validation_mode": "strict",
 "input_schema": [
   {"name": "cnpj", "type": "string", "location": "body", "required": true},
   {"name": "uf", "type": "string", "location": "body", "required": false, "default": ""}]}
```

## Observações para o piloto de margem

1. **Não é o gatilho do piloto** — o piloto é agendado (`cron`). Mas o webhook é a **alternativa
   orientada a evento** se o ERP puder notificar pedido novo: trocaria polling de 5 minutos por
   disparo no fato, o que elimina de uma vez a janela incremental e a espera de até 5 minutos.
   `[LACUNA]` Não se sabe se o ERP consegue emitir webhook — é pergunta de ERP, não de
   plataforma.

2. **O at-least-once do `immediately` é o argumento mais forte já registrado a favor de
   idempotência (L6)** e não depende do gatilho escolhido: se um dia o piloto virar webhook, a
   escrita de preço precisa tolerar reexecução do mesmo payload. Peça combinável:
   `bulk_insert` com `load_strategy: upsert` + `merge_keys`, e `on_update: never` /
   `keep_if_empty` nas colunas que não devem ser tocadas (ver `nos/bulk-insert.md`).

3. **`on_finish` tem 429 sob carga; `immediately` não.** Se algum dia um webhook de preço for
   chamado em rajada, `on_finish` recusa e `immediately` absorve — mas paga com reexecução.
   Escolha consciente entre perder disparo e repetir escrita.

4. **Use o Cofre de Segredos desde o primeiro dia.** O contrato do MCP diz literalmente para não
   usar valor literal em fluxo novo. Vale mais no piloto porque o fluxo será exportado/versionado.

5. **`input_schema` com `validation_mode: strict`** é a única validação de contrato de entrada
   encontrada em qualquer gatilho — `manual` explicitamente **não** valida schema
   (`nos/manual-(gatilho).md § Limites`), e `cron` não recebe dados. Se o piloto precisar validar
   entrada, o webhook é o único gatilho que faz isso nativamente.
