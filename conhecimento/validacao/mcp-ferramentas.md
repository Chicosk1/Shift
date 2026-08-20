# Inventário do MCP do Shift — dump real

> **Status: CONECTADO E INVENTARIADO** — 2026-08-20 (Fase 0).
> Fonte: introspecção do servidor `shift` em `https://shift.viasoftcloud.com.br/mcp`.
> Marcador: `[CONFIRMADO-MCP]` para tudo neste arquivo.
>
> **48 ferramentas expostas** — contra **15** descritas na documentação. A doc
> (`agente-shift-funcionamento.md`, datada de 2026-04-20) está substancialmente defasada.
> Ver `divergencias.md`.

---

## 1. Conexão

| Item | Valor |
|---|---|
| Servidor | `shift` |
| Transporte | `http` (Streamable HTTP) |
| Endpoint | `https://shift.viasoftcloud.com.br/mcp` |
| Autenticação | header `Authorization: Bearer <API KEY>` |
| Arquivo de config | `C:\Shift\.mcp.json` (na **raiz** — obrigatório) |
| Ferramentas carregadas | 48 |

### Duas pegadinhas de setup

**1. O `.mcp.json` tem de estar na raiz do projeto.** O `claude mcp add --scope project`
reportou gravar em `C:\Shift\.mcp.json`, mas o arquivo apareceu depois em
`C:\Shift\.claude\.mcp.json` (mesmo tamanho e mtime — foi movido, não reescrito). Nessa
localização o Claude Code **não detecta o servidor** e nem exibe o prompt de aprovação. Se o
servidor "desaparecer", conferir a localização do arquivo antes de qualquer outra hipótese.

**2. Um GET pelo navegador no endpoint devolve erro — e isso é o comportamento correto.**
`{"code":-32600,"message":"Not Acceptable: Client must accept text/event-stream"}` é um servidor
MCP Streamable HTTP recusando requisição sem o header `Accept: text/event-stream`. Serve como
teste de vida do endpoint, não como sinal de falha.

> **Segurança:** o `.mcp.json` guarda a API Key em texto puro na raiz do repositório. Está no
> `.gitignore` (verificado com `git check-ignore` nas duas localizações possíveis) e busca por
> fragmento do token confirma que **a chave nunca entrou em commit**. Recomenda-se rotação.

---

## 2. As 48 ferramentas

Convenção da própria plataforma, declarada nas descrições: operação destrutiva **exige
aprovação humana**; ferramenta read-only não passa pelo nó de approval.

### A. Leitura — plataforma, conexões e webhooks (8)

| Ferramenta | O que faz | Parâmetros (**obrigatório**) |
|---|---|---|
| `list_projects` | Projetos do workspace atual visíveis ao usuário | `limit` |
| `get_project` | Detalhes do projeto: nº de workflows e conexões | **`project_id`** |
| `list_project_members` | Membros e roles (`EDITOR` ou `CLIENT`) + data de ingresso | **`project_id`** |
| `list_connections` | Conexões visíveis (nome, tipo, host). Nunca retorna senha | `project_id` |
| `get_connection` | Metadados não-sensíveis: nome, tipo, host, porta, banco, usuário | **`connection_id`** |
| `test_connection` | SUCESSO/FALHA com mensagem do banco. Abre conexão real | **`connection_id`** |
| `diagnosticar_conexao` | Falha **por etapas** (DNS, TCP, handshake, login) com causa provável; considera relay | **`connection_id`** |
| `list_webhooks` | Nós webhook dos workflows, com path e workflow associado | `project_id` |

### B. Leitura — definição de fluxo (6)

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `list_workflows` | Workflows com nome, status e id | `project_id`, `limit` |
| `get_workflow` | Nome, status, nº de nós, se é template/publicado, **parâmetros de entrada** | **`workflow_id`** |
| `ler_estado_do_fluxo` | **Definição completa**: nós com config, arestas com handles, variáveis, io_schema | **`workflow_id`** |
| `ler_campos_do_no` | Colunas no output de um nó. `source` = `execution` / `inferred` / `unknown` | **`workflow_id`**, **`node_id`**, `include_sample` |
| `avaliar_fluxo` | Check-up determinístico: config inválida, nós órfãos, **SQL destrutivo sem proteção**, caminho de erro solto | **`workflow_id`** |
| `ler_documentacao` | Documentação markdown do fluxo | **`workflow_id`** |

### C. Leitura — execuções (4)

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `list_recent_executions` | Últimas execuções com status e timestamps | **`workflow_id`**, `limit` |
| `get_execution_status` | `RUNNING`/`COMPLETED`/`FAILED`/`CANCELLED`/`CRASHED` + erro | **`execution_id`** |
| `ler_execucao` | Passo a passo: por nó, status, duração, linhas in/out, erro, rejeições agrupadas. Um nível por chamada | **`execution_id`** |
| `ler_rejeicoes` | **Dead-letter**: grupos por causa, e amostras com campo culpado e valor. Campos sensíveis mascarados | **`execution_id`**, `causa`, `node_id`, `limite` |

### D. Leitura — catálogo de nós (2)

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `list_nodes` | Índice dos tipos de nó com categoria e descrição | `category`, `query` |
| `describe_node` | **Contrato completo**: campos de config, transforms, exemplos. Aceita alias | **`node_type`** |

### E. Leitura — bases internas (1)

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `listar_bases_internas` | Bases internas com id, nome e colunas. Resolve destino por nome | — |

### F. Escrita direta — exige aprovação humana (10)

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `create_project` | Cria projeto. Requer role `MANAGER` no workspace | **`name`**, `description` |
| `create_workflow` | Cria workflow vazio (draft). Requer `CONSULTANT` no ws + `EDITOR` no projeto | **`name`**, `project_id`, `description` |
| `add_node` | Adiciona nó ao workflow | **`workflow_id`**, **`node_type`**, **`position`**, `config` |
| `add_edge` | Conecta dois nós | **`workflow_id`**, **`source_id`**, **`target_id`**, `source_handle`, `target_handle` |
| `remove_node` | Remove nó **e todas as arestas em cascata**. Irreversível | **`workflow_id`**, **`node_id`** |
| `remove_edge` | Remove aresta por `edge_id` | **`workflow_id`**, **`edge_id`** |
| `update_node_config` | Patch **shallow** na config do nó | **`workflow_id`**, **`node_id`**, **`config_patch`** |
| `set_workflow_variables` | **Substitui integralmente** a lista de variáveis | **`workflow_id`**, **`variables`** |
| `criar_base_interna` | Cria base interna. Tipos: text, long_text, number, integer, date, datetime, boolean, email, phone, cpf, cnpj | **`nome`**, **`colunas`**, `descricao` |
| `escrever_documentacao` | **Substitui** a documentação inteira. Sem desfazer; usuário vê o diff | **`workflow_id`**, **`conteudo`** |

### G. Execução — exige aprovação humana (3)

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `execute_workflow` | Dispara execução. Ver `get_workflow` para os parâmetros exigidos | **`workflow_id`**, `trigger_params` |
| `trigger_webhook_manually` | Simula chamada ao webhook em modo teste | **`workflow_id`**, `payload` |
| `cancel_execution` | Solicita cancelamento. Assíncrono, não garante efeito imediato | **`execution_id`** |

### H. Build session `pending_*` — aprovação em lote no confirm (10)

Modelo de trabalho distinto: as peças são propostas como *ghost nodes* numa sessão e o usuário
**aprova tudo de uma vez**, em vez de aprovar nó por nó. É o caminho indicado para construir
fluxo — não a família `add_node`/`add_edge` direta.

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `pending_add_node` | Nó pendente com `temp_id`. `parent_temp_id` coloca a etapa **dentro do corpo de um loop** (exige `body_mode='inline'`) | **`session_id`**, **`temp_id`**, **`node_type`**, **`label`**, `config`, `position`, `parent_temp_id` |
| `pending_add_nodes_batch` | Vários nós de uma vez — **preferível** a chamadas repetidas | **`nodes`** |
| `pending_add_edge` | Liga `temp_id` desta sessão **ou** id real de nó existente | **`session_id`**, **`source_temp_id`**, **`target_temp_id`**, handles |
| `pending_add_edges_batch` | Todas as arestas de uma vez — **preferível** | **`edges`** |
| `pending_remove_node` | Descarta nó pendente e suas arestas | **`session_id`**, **`temp_id`** |
| `pending_remove_edge` | Descarta aresta por `edge_id` ou par de `temp_id` | **`session_id`** + identificação |
| `pending_update_node` | Patch shallow na config pendente | **`session_id`**, **`temp_id`**, **`config_patch`** |
| `pending_set_variables` | **Substitui** as variáveis pendentes. Tipos incluem `table_reference`, `connection`, `file_upload`, `secret` | **`session_id`**, **`variables`** |
| `pending_set_io_schema` | Schema de I/O do subfluxo. **Substitui as duas listas** — mandar `inputs` E `outputs` | **`session_id`**, `inputs`, `outputs` |
| `pending_preencher_conexao` | **Propõe** valores do formulário de conexão num cartão para aprovação. **Senha não pode ser proposta** | **`campos`**, `confirmado_pelo_usuario` |

### I. Interação e diagnóstico de rede local (4)

| Ferramenta | O que faz | Parâmetros |
|---|---|---|
| `planejar` | Apresenta plano em passos antes de construir. Read-only no canvas | **`passos`**, `resumo` |
| `perguntar` | Perguntas de clarificação com opções clicáveis; **pausa** até resposta. Máx. 5 | `perguntas` / `pergunta` + `opcoes` |
| `comando_para_maquina_local` | Devolve o comando a rodar na máquina do cliente e em qual máquina. **Não executa** | **`objetivo`** + contexto |
| `ler_saida_colada` | Faz parse da saída que a pessoa colou. Formato desconhecido volta como NÃO RECONHECIDA | **`saida`**, `objetivo` |

Objetivos de `comando_para_maquina_local`: `mssql_listener_ports`, `mssql_databases`,
`windows_sql_instances`, `windows_sql_tcp_port`, `windows_enable_tcp_static_port` (**escrita**),
`probe_destination_reachable`, `relay_log_tail`, `relay_diag_bundle`, `relay_destinos`.

---

## 3. Verificação executada

Uma única chamada de estado, read-only: `list_projects` → *"Nenhum projeto encontrado no
workspace."* Resposta válida — o workspace alcançado pela chave não tem projetos, ou o workspace
"atual" não é o `Treinamento` das aulas. **A confirmar na Fase 2.**

`list_nodes` também foi chamada (read-only) e devolveu **63 tipos de nó** — ver
`mcp-catalogo-nos.md`.

**Nenhuma ferramenta de escrita, de execução ou `pending_*` foi executada.**

---

## 4. O que isto muda no projeto

1. **O MCP constrói fluxo.** Havia registro em `lacunas.md` (L13) de que provavelmente não —
   baseado só na doc. **Errado.** Existem `create_workflow`, `add_node`, `add_edge`,
   `update_node_config` e a família `pending_*` inteira.
2. **O caminho indicado para construir é a build session `pending_*`**, com aprovação em lote,
   não as ferramentas diretas.
3. **`avaliar_fluxo` detecta "SQL destrutivo sem proteção" e "caminho de erro solto".** É um
   checklist de pré-produção que já existe na plataforma — a Fase 4 deve integrá-lo ao
   checklist da skill em vez de reinventá-lo.
4. **`ler_rejeicoes` é a trilha de dead-letter** pedida em §5.6 do plano, e já mascara campos
   sensíveis.
5. **`comando_para_maquina_local` + `ler_saida_colada`** formam um fluxo de diagnóstico de rede
   guiado, com foco em SQL Server — diretamente relevante para conectar o ERP.
6. **`perguntar` e `planejar` são ferramentas de protocolo**, não de dados: a plataforma tem
   opinião sobre pausar para perguntar e planejar antes de construir. A Fase 4 deve codificar
   isso nas convenções.
