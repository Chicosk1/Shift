---
title: Webhook (gatilho)
---

**Categoria:** Gatilho
**Tipo interno:** `webhook`

## Descrição

Expõe uma URL HTTP pública que, ao receber uma chamada, inicia o workflow imediatamente com o payload recebido. Permite integrar o Shift a qualquer sistema externo que suporte envio de webhooks — ERPs, plataformas de e-commerce, ferramentas de automação, etc.

Cada nó Webhook gera duas URLs distintas:

| URL | Quando usar |
| --- | --- |
| **Test URL** (`/api/v1/webhook-test/{path}`) | Testes durante desenvolvimento; o workflow roda em modo draft |
| **Production URL** (`/api/v1/webhook/{path}`) | Uso real; disponível somente com o workflow em **Produção** e **Publicado** |

## Saída produzida

```
{
  "trigger_type": "webhook",
  "status": "triggered",
  "http_method": "POST",
  "headers": { "content-type": "application/json", ... },
  "query_params": { "source": "erp" },
  "data": { ...body da requisição... }
}
```

O body da requisição fica no campo definido por `output_field` (padrão `data`). Query strings ficam em `query_params` e os cabeçalhos em `headers`.

## Configurações

### Aba Parameters

| Campo | Tipo | Padrão | Descrição |
| --- | --- | --- | --- |
| `http_method` | enum | `POST` | Método HTTP aceito: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD` |
| `path` | string | UUID gerado automaticamente | Sufixo da URL do webhook. Gerado na primeira abertura; pode ser personalizado |
| `authentication` | objeto | `none` | Mecanismo de autenticação (ver abaixo) |
| `respond_mode` | enum | `immediately` | Quando o Shift responde ao chamador (ver abaixo) |
| `output_field` | string | `data` | Campo da saída onde o body da requisição é gravado |

#### Autenticação

| Tipo | Campos adicionais | Descrição |
| --- | --- | --- |
| `none` | — | Sem autenticação (qualquer chamada é aceita) |
| `bearer` | `bearer_token` | Exige `Authorization: Bearer <token>`. Informe apenas o token — o prefixo `Bearer ` é do protocolo e não deve ser digitado. O nome do esquema é aceito sem diferenciar maiúsculas |
| `header` | `header_name`, `header_value` | Exige um header com valor secreto (ex.: `X-Webhook-Secret`) |
| `basic` | `username`, `password` | HTTP Basic Auth |
| `jwt` | `jwt_secret`, `jwt_algorithm` | Bearer token JWT; algoritmos suportados: HS256, HS384, HS512, RS256 |

#### Segredo vindo do Cofre (recomendado)

Em vez de digitar o token no nó, o webhook pode **apontar** para um segredo
cadastrado no **Cofre de Segredos** (campo `secret_id`). O que muda:

- **O segredo sai de dentro do fluxo.** Quem abre ou exporta o desenho vê que
existe uma proteção, não qual é.

- **Trocar o token não exige mexer no fluxo.** A troca é uma rotação no cofre; o
nó continua apontando para o mesmo lugar. Durante a janela de graça da rotação
o token antigo ainda é aceito, então a integração não cai enquanto o outro lado
não atualiza.

- **Revogar vale na hora**, em todos os fluxos que usam aquele segredo. Isso é o
comportamento desejado numa emergência — e é bom saber disso antes de revogar
um segredo de rotina, porque ele pode estar em mais de uma integração.

Cada tipo pede um **modo** de segredo:

| Tipo | Modo do segredo | Por quê |
| --- | --- | --- |
| `bearer`, `header`, `basic` | `verify` | O Shift só precisa **comparar** o que chegou com o que guardou |
| `jwt` | `reveal` | Conferir a assinatura HS256 exige o **valor original** do segredo |

Um webhook `jwt` apontando para um segredo `verify` é recusado ao entrar em
Produção, com a mensagem dizendo o que fazer — não vira erro obscuro em execução.

Os campos que **não são segredo** continuam no nó: `header_name` (nome do
header) e `username` (no Basic, o segredo é a senha).

> **Os fluxos que já rodam com o valor digitado continuam funcionando**, sem
> nenhuma alteração. A referência é uma alternativa nova, não uma substituição
> forçada. Se os dois estiverem preenchidos, a **referência vence** — o valor
> digitado passa a ser ignorado.

Ao publicar (entrar em **Produção**), o Shift confere a referência: segredo
inexistente, revogado, vencido, de outro Espaço ou de modo incompatível barra a
publicação com a explicação do que corrigir. Já em produção, uma chamada com
segredo inválido — inclusive segredo revogado depois — recebe `401` e aparece na
trilha de chamadas recebidas como falha de autenticação. A recusa é sempre a
mesma mensagem genérica, de propósito: dizer "esse segredo está revogado" para
quem chama a rota pública seria entregar informação a quem está tentando adivinhar.

#### Modo de resposta (Respond)

| Valor | Comportamento |
| --- | --- |
| `immediately` | Responde `200 OK` assim que o payload é recebido, sem esperar o workflow terminar |
| `on_finish` | Mantém a conexão aberta e responde somente quando o workflow conclui |
| `using_respond_node` | A resposta é controlada por um nó "Respond to Webhook" no fluxo |

#### O "aceitei" do modo `immediately` sobrevive a reinício

Nesse modo o Shift **grava a chamada numa fila durável** e responde na hora;
quem executa o fluxo é um processo separado, logo em seguida. Antes o disparo
existia só na memória: se a plataforma reiniciasse naquele instante (deploy,
queda), ele se perdia em silêncio — e quem chamou já tinha recebido o `200`.
Agora um reinício não perde nada, porque a chamada já está registrada.

O preço dessa garantia é que o fluxo **pode rodar duas vezes**. Se a execução
for interrompida no meio, o disparo volta para a fila e é executado de novo — o
Shift prefere repetir a perder. Se o seu fluxo grava em sistemas externos e não
é *idempotente* (rodar duas vezes com o mesmo payload produz dois pedidos, dois
e-mails, dois lançamentos), trate isso dentro do fluxo: use uma chave do payload
para verificar se aquele registro já foi processado antes de gravar. O histórico
de execuções mostra quantas vezes o fluxo realmente rodou para cada chamada.

Isso vale só para `immediately`. No `on_finish` o fluxo roda inline, porque a
resposta É o resultado dele.

### Aba Options (avançado)

| Campo | Padrão | Descrição |
| --- | --- | --- |
| `response_code` | `200` | Código HTTP retornado ao chamador |
| `response_data` | `first_entry_json` | O que incluir no corpo da resposta: primeiro registro JSON, todos os registros, ou sem corpo |
| `raw_body` | `false` | Recebe o body sem parse JSON (útil para payloads binários ou texto puro) — indisponível para GET/HEAD |
| `allowed_origins` | *(vazio)* | Origens permitidas para CORS (`*` ou domínio específico) |

## Como testar

1. Abra o nó no editor e clique em **Listen for test event**.

2. O Shift fica aguardando por até 120 segundos.

3. Dispare uma requisição para a **Test URL** exibida no painel.

4. O payload capturado é injetado automaticamente na execução de teste.

## Limites e guardrails

- A **Production URL** só fica acessível após o workflow ser colocado em Produção e publicado.

- `path` vazio → o ID do workflow é usado como path de fallback.

- Com `respond_mode: on_finish`, chamadores com timeout curto podem receber erro de conexão encerrada antes de o workflow concluir.

- `raw_body: true` com método GET ou HEAD é bloqueado pela UI (GET/HEAD não carregam body).

- **Plataforma lotada, com `respond_mode: on_finish` → `429 Too Many Requests`
imediato**, com o header `Retry-After: 30`. Quando o teto de execuções
simultâneas (global ou do projeto) está atingido, a chamada é recusada na
hora, sem enfileirar — configure o sistema chamador para reenviar após o
`Retry-After`. A recusa aparece na auditoria de chamadas recebidas como
**Rejeitado: plataforma lotada**, com o conteúdo recebido.

- No `respond_mode: immediately` **não existe mais esse 429**: a chamada não
disputa vaga de execução, ela entra na fila durável. Uma rajada é aceita e
guardada em vez de recusada — recusar seria perder um disparo que a fila
consegue absorver. O trabalho de fundo tem orçamento próprio de execução, por
isso um pico de webhooks não tira a vaga de quem está usando a plataforma na
tela.

- Se nem gravar a chamada na fila for possível (banco indisponível), a resposta
é `503` — o Shift não responde "aceitei" para algo que não conseguiu
registrar.

## Observabilidade

A saída inclui `http_method`, `headers` e `query_params`, visíveis no painel de execução para depurar problemas de integração.
