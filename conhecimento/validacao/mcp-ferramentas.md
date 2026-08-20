# Inventário do MCP do Shift

> **Status: REGISTRADO NO CAMINHO CORRETO — AGUARDANDO APROVAÇÃO** — atualizado em 2026-08-20 (Fase 0).
> Este arquivo é um **gabarito derivado da documentação**, não um dump real.
> Ele deve ser substituído pelo dump real assim que o servidor for aprovado e carregado.

---

## 1. Situação

O servidor foi registrado no escopo correto via `claude mcp add --scope project`, que gravou
`C:\Shift\.mcp.json`. **Mas as ferramentas ainda não carregam**, por dois motivos somados:

1. Servidor declarado em `.mcp.json` exige **aprovação explícita por projeto**. Hoje
   `enabledMcpjsonServers` está `[]` em `~/.claude.json` — nem aprovado, nem recusado.
2. A sessão em curso foi aberta antes do registro. É preciso **reabrir a sessão** e aceitar o
   prompt de aprovação do servidor `shift`.

**Pegadinha encontrada no caminho:** o `claude mcp add` reportou gravar em
`C:\Shift\.mcp.json`, mas o arquivo apareceu depois em `C:\Shift\.claude\.mcp.json` (mesmo
tamanho e mtime — foi movido, não reescrito). Nessa localização o Claude Code **não o detecta**,
e por isso nem chega a exibir o prompt de aprovação. Corrigido movendo de volta para a raiz.
Se o servidor "desaparecer" no futuro, conferir a localização do arquivo antes de qualquer outra
hipótese.

**Diagnóstico útil:** um GET pelo navegador em `https://shift.viasoftcloud.com.br/mcp` responde
`{"code":-32600,"message":"Not Acceptable: Client must accept text/event-stream"}`. Isso **não é
falha** — é um servidor MCP Streamable HTTP recusando corretamente uma requisição sem o header
`Accept: text/event-stream`. Serve para confirmar que o endpoint está vivo.

| Item | Valor |
|---|---|
| Nome do servidor | `shift` |
| Transporte | `http` |
| Endpoint | `https://shift.viasoftcloud.com.br/mcp` |
| Autenticação | header `Authorization: Bearer <API KEY>` |
| Arquivo | `C:\Shift\.mcp.json` (241 bytes) |
| Escopo | projeto `C:\Shift` — correto |
| Aprovação | **pendente** (`enabledMcpjsonServers: []`) |

**Nada foi executado contra o MCP** — nem leitura, nem escrita, nem chamada HTTP ao endpoint.

> ⚠️ **`.mcp.json` contém a API Key em texto puro** e fica na raiz do repositório. Foi
> adicionado ao `.gitignore` antes de qualquer commit — verificado com `git check-ignore` e com
> busca por fragmento do token em todo o histórico. **A chave nunca entrou em commit.**
> Ainda assim, a chave já circulou em terminal e em captura de tela: **recomenda-se rotação**.

### O que falta

Reabrir a sessão em `C:\Shift` e **aprovar o servidor `shift`** quando o prompt aparecer.
Depois disso, regerar este arquivo com o dump real e preencher as colunas `[LACUNA]`.

---

## 2. Gabarito esperado — 15 ferramentas

Fonte: `conhecimento/bruto/docs/documentacao-tecnica/agente-shift-funcionamento.md` §4
(documento datado de 2026-04-20, fases 0–7). Marcador: `[CONFIRMADO-DOC]` para nome e
descrição; `[LACUNA]` para os parâmetros, que a doc não especifica.

A doc afirma que "cada tool tem schema JSON e validação de permissão embutida", mas **não
publica os schemas**. Por isso a coluna de parâmetros está vazia de propósito — preenchê-la
por dedução seria exatamente o tipo de invenção que esta base de conhecimento existe para
evitar.

### Workflow (5)

| Ferramenta | Descrição (doc) | Escrita? | Parâmetros |
|---|---|---|---|
| `list_workflows` | Lista do projeto com filtros | Leitura | `[LACUNA]` |
| `get_workflow_details` | Detalhes + nós + última execução | Leitura | `[LACUNA]` |
| `execute_workflow` | Executa um workflow | **⚠️ DESTRUTIVO — requer approval** | `[LACUNA]` |
| `list_executions` | Histórico de execuções | Leitura | `[LACUNA]` |
| `get_execution_details` | Logs, dead-letters, duração | Leitura | `[LACUNA]` |

### Project (3)

| Ferramenta | Descrição (doc) | Escrita? | Parâmetros |
|---|---|---|---|
| `list_projects` | Projetos visíveis ao usuário | Leitura | `[LACUNA]` |
| `get_project_details` | — (doc não detalha) | Leitura | `[LACUNA]` |
| `list_project_members` | — (doc não detalha) | Leitura | `[LACUNA]` |

### Connection (3)

| Ferramenta | Descrição (doc) | Escrita? | Parâmetros |
|---|---|---|---|
| `list_connections` | — (doc não detalha) | Leitura | `[LACUNA]` |
| `get_connection_details` | Metadados **sem credenciais** | Leitura | `[LACUNA]` |
| `test_connection` | Valida conectividade | Leitura (efeito de rede) | `[LACUNA]` |

### Webhook (3)

| Ferramenta | Descrição (doc) | Escrita? | Parâmetros |
|---|---|---|---|
| `list_webhooks` | — (doc não detalha) | Leitura | `[LACUNA]` |
| `get_webhook_details` | URLs de test/prod | Leitura | `[LACUNA]` |
| `list_webhook_executions` | — (doc não detalha) | Leitura | `[LACUNA]` |

### Utilitários (1)

| Ferramenta | Descrição (doc) | Escrita? | Parâmetros |
|---|---|---|---|
| `search_global` | Busca cross-entity | Leitura | `[LACUNA]` |

---

## 3. Regras de uso a observar quando conectar

`[CONFIRMADO-DOC]`, mesma fonte:

- **`execute_workflow` é a única ferramenta destrutiva do conjunto.** As outras 14 são de
  leitura. A doc é explícita: "tools read-only **não** passam pelo nó de approval. Só
  mutações/execuções."
- **Não existe ferramenta de criação/edição de fluxo no MCP.** Nenhuma tool de `create_*`,
  `update_*` ou `delete_*` é listada. Isso tem consequência direta para as fases seguintes:
  a construção de fluxos provavelmente **não** é automatizável pelo MCP, só a inspeção e a
  execução. **Confirmar contra o dump real** — se a doc estiver defasada, muda a estratégia
  da Fase 5.
- **A API Key limita o conjunto por whitelist** (`allowed_tools`, com wildcard `*`). Se o dump
  real vier com menos de 15 ferramentas, a causa provável é a whitelist da chave, não ausência
  na plataforma.
- Chaves têm prefixo documentado `shk_`, hash Argon2, exibição do valor em claro **uma única
  vez**, expiração de 30/60/90 dias ou nunca, e são emitidas em **Projeto → Chaves de API**.
- Arquitetura por trás: LangGraph StateGraph de 6 nós
  (guardrails → intent → planner → approval → executor → report), com auditoria em
  `ai_agent_audit_log`.

---

## 4. A conferir contra o dump real

Lista de verificação para a próxima sessão:

1. O número de ferramentas é 15? Se menor, é whitelist da chave?
2. Os 15 nomes coincidem com este gabarito?
3. Existe alguma ferramenta de **escrita de definição de fluxo** não documentada?
4. Qual o schema real de parâmetros de cada uma? (preencher as colunas `[LACUNA]`)
5. **D6 —** o prefixo da chave em uso não é o `shk_` documentado. Divergência de doc ou tipo
   de credencial diferente?
6. Registrar tudo o que divergir em `conhecimento/validacao/divergencias.md`.
