# Nó `manual` — Trigger Manual

| | |
|---|---|
| **Tipo (MCP)** | `manual` |
| **Rótulo na interface** | Manual / Trigger Manual |
| **Categoria** | `trigger` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — (não afetado pela consolidação) |

## O que faz

`[CONFIRMADO-MCP]` Inicia o workflow quando acionado manualmente pelo botão **Play** no editor.
Repassa o `input_data` recebido para o restante do fluxo. Suporta payload de fallback configurado
no próprio nó. — `describe_node('manual')`

## Quando usar

`[CONFIRMADO-MCP]`
- Disparar o workflow manualmente para testes ou execuções pontuais.
- Receber um payload via botão Play no editor.

`[CONFIRMADO-DOC]` Também dispara **por API**, com o `input_data` exposto em `data`.
— `guias-de-uso/nos/manual-(gatilho).md`

## Parâmetros de configuração

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `payload` | object | não | Payload de fallback, usado **quando o `input_data` externo está vazio** |

`[CONFIRMADO-MCP]` — `describe_node('manual')`

`[UI-OBSERVADA]` Na interface o campo aparece como **"Payload de teste"** e exige JSON válido.
— `m3-2A`

## Entradas esperadas

**Nenhuma.** `[VÍDEO]` É a regra de forma dos gatilhos: *"o único nó que só tem saída é o
manual, o resto tem entrada e saída"*. — `m1:73`
`[CONFIRMADO-DOC]` `conceitos.md § Nó` generaliza para todos os gatilhos.

## Saídas produzidas

`[CONFIRMADO-MCP]` O `input_data` recebido, repassado adiante. Sem `input_data`, o `payload`
configurado.

## Erros comuns

`[LACUNA]` Nenhum erro específico é declarado no contrato do MCP nem observado em aula.
Falta descobrir o comportamento quando `payload` contém JSON inválido — a interface valida
`[UI-OBSERVADA]`, mas não se sabe o que acontece via API.

## Exemplo

`[CONFIRMADO-MCP]` Config do próprio contrato, para a pergunta *"Como testar um workflow com
dados de exemplo sem chamar a API?"*:

```json
{"payload": {"nome": "Teste", "valor": 100}}
```

`[VÍDEO]` Em aula é o primeiro nó do fluxo `Importar Funcionários`, adicionado sem nenhuma
configuração — só para dar início ao fluxo. — `m1:65`

## Observação para o piloto

Não serve como gatilho do fluxo de ajuste de margem, que é agendado (`cron`). Serve para
**testar** o fluxo antes de publicar, já que `cron` só dispara com o fluxo publicado e em modo
Produção `[CONFIRMADO-DOC]` (`nos/agendamento-(cron).md`). Ver `lacunas.md` L2.
