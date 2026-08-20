---
title: Controle de Acesso
---

> **Status: implementado** (branch `feat/access-redesign`). Backend, frontend
> e testes refletem este desenho. Pendência operacional: a **migração
> `c7d8e9f0a1b2` é destrutiva** (dropa `client_members`, remove convites
> `CLIENT`) e ainda **não foi aplicada** — exige rollout coordenado e
> re-provisão de Observadores. Ver [Estado de implementação](https://shift.viasoftcloud.com.br/docs/technical/access-control#estado-de-implementa%C3%A7%C3%A3o).

Este documento descreve a hierarquia de papéis, a regra de herança e o
que cada papel pode fazer na plataforma Shift.

## Princípio condutor

As telas e as rotas devem responder a **duas perguntas do usuário**, não
espelhar tabelas do banco:

1. *Quem pode acessar este recurso, e por quê?* (visão por recurso, com
herança explicada)

2. *O que esta pessoa pode acessar?* (visão por pessoa)

E há **dois públicos distintos** que nunca se confundem:

- **Consultor Viasoft** — opera a plataforma (org/workspace).

- **Cliente final** — apenas **acompanha** um projeto (Observador) ou
preenche um formulário via link. Nunca administra nada.

## Hierarquia

```
Organização > Workspace > Projeto
                 │
                 └── Cliente (grupo econômico) é uma ENTIDADE de
                     organização do trabalho, não um nível de acesso.
```

- `Organização` é o nível mais alto (a conta Viasoft).

- `Workspace` agrupa o trabalho dos consultores.

- `Projeto` é o contrato/iniciativa. É o nível onde o **cliente final**
pode ser convidado como **Observador** (read-only).

- `Cliente` continua existindo como entidade (grupo econômico,
estabelecimentos/CNPJ), mas **não tem mais membership de acesso**. O
acesso externo se dá no **projeto**.

## Papéis

### `OrganizationRole`

| Papel | O que pode fazer |
| --- | --- |
| `OWNER` | Dono da organização (único). Billing, transferência, deletar. **Único que acessa todos os espaços** (herda ADMIN em todos). |
| `ADMIN` | Gerencia workspaces, membros e catálogo da org (poder administrativo). **Não dá acesso a espaços por si só** — para *trabalhar dentro* de um espaço precisa de um `WorkspaceMember` explícito (que ele mesmo pode se conceder). |
| `MEMBER` | Faz parte da organização sem privilégios administrativos. **Não dá acesso a espaços por si só** — o acesso a cada Espaço vem de um `WorkspaceMember` explícito. |

### `WorkspaceRole`

O que separa os três degraus é um **princípio explícito**:

| Papel | Princípio | Engloba |
| --- | --- | --- |
| `VIEWER` | Lê tudo, muda nada | ver clientes, projetos, workflows, execuções, logs (sem segredos); testar conexão |
| `EDITOR` | Dono do **trabalho de consultoria** | criar/editar/**deletar** cliente, projeto, workflow, conexão, execução; **executar em não-produção**; ferramentas de autoria; provisionar Observadores; instâncias/links de formulário |
| `ADMIN` | Governa o **workspace em si** | membros, convites, chaves de API, webhooks, templates compartilhados, storage; **executar em produção**; deletar o workspace |

**Bonus:** `WorkspaceMember.is_owner` (bool) — apenas 1 owner por
workspace, com privilégio adicional de transferir a propriedade
(`PUT /workspaces/{id}/owner`) e exibido com ícone de coroa. A remoção
do membership do owner só é permitida após transferir a propriedade.

### `Projeto` → Observador (cliente final)

| Papel | O que pode fazer |
| --- | --- |
| `OBSERVADOR` | **Somente leitura** do andamento/status/execuções de **um** projeto específico. Não vê outros projetos, não vê segredos, não edita nada. |

O Observador é um **usuário convidado/nomeado** (login), provisionado
pelo consultor, vinculado a **um projeto**. Para dar acesso a mais de
uma pessoa ou a outro projeto, o consultor provisiona cada grant.

Os **formulários** do cliente não dependem deste papel: são **links
tokenizados** (`/f/{token}`) gerados pelo consultor.

## Gate de produção

`EDITOR` executa workflows livremente contra ambientes de
**não-produção**. Executar um workflow que toca uma conexão marcada como
**produção** exige `ADMIN`.

- O gate se baseia no campo `Connection.environment`, que passa a ser
**obrigatório** e confiável (idealmente imutável após a criação ou
alterável só por ADMIN, com auditoria).

- A checagem inspeciona as conexões referenciadas pelo workflow no
momento da execução; se qualquer uma for de produção e o usuário não
for ADMIN do workspace → `403`.

## Regras de herança

- **Só o Org `OWNER`** herda `ADMIN` em todos os workspaces da org (vê e entra
em tudo).

- Org `ADMIN` e `MEMBER` **não herdam** acesso a espaços. O papel na org é
poder administrativo da organização (gerenciar membros, criar espaços,
catálogo); o acesso a *trabalhar dentro* de um espaço vem **exclusivamente**
de um `WorkspaceMember` explícito. Isso separa "administrar a org" de
"trabalhar no espaço" e permite escopar qualquer usuário (inclusive Admin)
a espaços específicos.

- `WorkspaceMember` direto define o papel; o Dono herda `ADMIN` (`source: "explicit"` para membership direta, `"inherited_org"` para o Dono).

- A visibilidade (navegação/listagem de espaços/projetos) segue a mesma regra:
só o Dono enxerga todos os espaços; ADMIN/MEMBER só os que têm membership
explícita (ou onde são Observador). A tela **Organização → Acesso** lista
todos os espaços para o Dono/Admin gerenciarem, via consulta administrativa
própria (independe da visibilidade de navegação).

### Gestão de acesso (telas e convite)

- **Organização → Acesso** é a visão macro "por pessoa": Dono/Admin gerenciam,
num lugar só, o papel na org e o acesso por espaço de cada pessoa
(endpoint `GET /organizations/{org}/access-overview`).

- **Convite multi-espaço**: um convite de escopo `ORGANIZATION` pode carregar
`workspace_grants` (`[{workspace_id, role}]`) — no aceite cria o
`OrganizationMember` + um `WorkspaceMember` por grant. Assim dá para já
definir, no convite, a quais espaços (e com que papel) a pessoa terá acesso.

### Cliente (entidade)

- Ler/escrever dados de um cliente segue a permissão de **workspace**
(VIEWER lê, EDITOR escreve). **Não há mais membership de cliente.**

### Projeto

1. Usuário tem `WorkspaceMember` (qualquer role efetivo)? → acesso
conforme o role do workspace (EDITOR/ADMIN editam, VIEWER lê).

2. Senão, é `ProjectObserver` deste projeto? → **somente leitura** deste
projeto.

3. Senão → `403`.

## Matriz de capacidades

### Organização

| Capacidade | MEMBER | ADMIN | OWNER |
| --- | --- | --- | --- |
| Ver org + membros | ✅ | ✅ | ✅ |
| Criar workspace · gerenciar membros · convidar | ❌ | ✅ | ✅ |
| Deletar org · billing · transferir | ❌ | ❌ | ✅ |

### Workspace (consultor)

| Capacidade | VIEWER | EDITOR | ADMIN |
| --- | --- | --- | --- |
| Ver clientes, projetos, workflows, execuções, logs | ✅ | ✅ | ✅ |
| Ver metadados de conexão · testar conexão | ✅ | ✅ | ✅ |
| Criar/editar/deletar cliente, projeto, estabelecimento | ❌ | ✅ | ✅ |
| Criar/editar/deletar workflow | ❌ | ✅ | ✅ |
| Criar/editar/deletar conexão | ❌ | ✅ | ✅ |
| Deletar execução | ❌ | ✅ | ✅ |
| Ferramentas de autoria (assistente código, preview, diagnose) | ❌ | ✅ | ✅ |
| **Executar workflow — não-produção** | ❌ | ✅ | ✅ |
| Provisionar/convidar Observador de projeto ¹ | ❌ | ✅ | ✅ |
| Criar instância de formulário + gerar link/token ¹ | ❌ | ✅ | ✅ |
| **Executar workflow — produção** (gate) | ❌ | ❌ | ✅ |
| Criar/editar/deletar template de input model | ❌ | ❌ | ✅ |
| Gerenciar membros do workspace + convidar p/ workspace | ❌ | ❌ | ✅ |
| API keys · webhooks subscriptions · storage | ❌ | ❌ | ✅ |
| Tela de Acesso (gerir papéis) | ❌ | ❌ | ✅ |
| Deletar workspace · transferir propriedade | ❌ | ❌ | ✅ (+owner) |

¹ Atribuído a `EDITOR` por serem parte de *tocar o projeto*, não de
administrar o workspace. Ajustável para `ADMIN` se a política mudar.

### Projeto (Observador — cliente final externo)

| Capacidade | OBSERVADOR |
| --- | --- |
| Ver andamento/status/execuções **deste** projeto | ✅ (leitura) |
| Preencher formulário | via link tokenizado |
| Ver outros projetos · conexões · segredos · editar | ❌ |

## Telas (arquitetura de informação)

```
Configurações
├── Organização › Acesso   (lista + papel efetivo + ação + convite inline)
└── Espaço › Acesso        (UMA tela: explícito + herdado, ação inline, convite)

Projeto
└── aba "Observadores"     (provisiona/convida/revoga espectadores do projeto)
```

- **Uma** tela de Acesso por nível, que mostra o papel **efetivo** + a
**origem** (`explícito`/`herdado da org`) e permite **agir ali mesmo**
(mudar papel, remover, convidar). Acaba a divisão "tela que mostra"
(matriz read-only) vs "tela que faz" (membros).

- Convite deixa de ser tela separada por escopo — vira **ação** dentro
da tela de Acesso.

- Observadores são geridos **dentro do projeto**, respondendo "quem
acompanha este projeto?".

## Enforcement

**Regra do projeto:** todo endpoint que lê ou escreve um recurso
armazenado **enforça a matriz na própria rota** (via
`require_permission` ou helper equivalente). Checagem feita só na camada
de serviço **não conta** como fonte de verdade. Isso é critério de
revisão de PR.

Hierarquia de checagem:

1. `require_permission(scope, required_role)` resolve `scope_id` via
request path/query/body.

2. `organization`/`workspace`: aplica a herança da org e compara com o
threshold.

3. `project`: tenta workspace (herança); senão verifica
`ProjectObserver` (read-only).

4. Ferramentas de autoria sem escopo no path (`/code-node/*`,
`/nodes/duckdb-preview`, `/connections/diagnose`) usam a dependência
`require_consultant`: exigem que o usuário seja **consultor** (tenha ao
menos uma membership de organização ou workspace). Isso bloqueia
Observadores externos — fechando o uso do `diagnose` como sonda de rede
interna — sem precisar de um `workspace_id` no payload. (Um gate por
`EDITOR` específico exigiria propagar escopo no corpo + frontend; fica
como evolução futura.)

5. `design-states` (`/workflows/{id}/design-states*`) não usam
`require_permission`, mas aplicam filtro multi-tenant próprio
(`_user_accessible_workflow_ids` / `_load_design_state_with_scope_check`).

## Convites

A plataforma é **100% por convite** — não há signup público. Fluxo
"link-first": o admin gera um convite e recebe um `invite_url` copiável.

### Escopos suportados

| Scope | Quem pode criar | Membership criada após aceite |
| --- | --- | --- |
| `ORGANIZATION` | Org `OWNER`/`ADMIN` | `OrganizationMember` |
| `WORKSPACE` | Workspace `ADMIN` | `WorkspaceMember` (+ `OrganizationMember MEMBER` se necessário) |
| `PROJECT` | Workspace `EDITOR`+ | `ProjectObserver` (read-only) |

O escopo `CLIENT` (membership de cliente) foi **removido** — substituído
por `PROJECT` (Observador). `InvitationScope.PROJECT` aponta para um grant
read-only externo (não um membership de projeto completo); o papel do
convite é sempre `OBSERVER`.

### Email é opcional (restritor de aceite)

Quando `email` é informado, apenas um usuário com esse email pode aceitar
(match case-insensitive). Quando `null`, qualquer usuário logado pode
aceitar — útil para links compartilhados.

### Regenerar link

`POST /invitations/{id}/regenerate-link` gera um **novo token**,
invalidando o anterior (`last_regenerated_at` registra auditoria). Não é
permitido regenerar convite `ACCEPTED`/`CANCELLED`.

### Primeiro acesso (Bootstrap)

Para o **primeiro usuário** (admin Viasoft), use `POST /api/v1/auth/bootstrap`
com `SHIFT_BOOTSTRAP_TOKEN` no ambiente. O backend cria User +
Organization + Workspace + memberships na mesma transação. **Remova** o
token após o uso (o endpoint retorna `403` sem a env var e `409` se já
houver qualquer usuário).

### Rate limiting

- Máximo 50 convites criados por usuário por hora.

- Máximo 50 convites `PENDING` por escopo.

- `/auth/register` herda o limite de 5 req/min do `slowapi`.

## Estado de implementação

Implementado na branch `feat/access-redesign`:

- [x] **Projeto/Observador**: `ClientMember`/`ClientRole` removidos;
`ProjectObserver` (read-only, escopo de projeto) criado. Migração
`c7d8e9f0a1b2` faz snapshot+drop de `client_members` e cria
`project_observers`.

- [x] **Convites**: `InvitationScope.CLIENT` → `PROJECT` (+ `project_id`,
drop de `client_id`); aceite cria `ProjectObserver`.

- [x] **Gate de produção**: `_enforce_production_gate` em
`/workflows/{id}/execute` e `/test` — conexão `production` exige
`workspace ADMIN`.

- [x] **Conexão (#3)**: editar/deletar conexão agora exige `EDITOR`
(uniforme).

- [x] **Workflow (#4)**: deletar workflow agora exige `EDITOR`.

- [x] **Ferramentas (#6)**: `/code-node/*`, `/nodes/duckdb-preview`,
`/connections/diagnose` exigem `require_consultant` (exclui Observador).

- [x] **Grupo 1 (#6)**: `POST .../versions`→EDITOR, `GET .../versions`→
VIEWER. `design-states` já aplicava filtro multi-tenant próprio.

- [x] **Formulários (#5)**: instância + token de formulário seguem o escopo
`client` (resolvido por herança de workspace) → `workspace EDITOR`.

- [x] **Telas**: tela única `espaco/acesso` (Membros + Matriz); aba
"Observadores" no projeto; grupo "Acesso dos Clientes" e telas de
convite separadas removidos.

Pendências operacionais (não-código):

- [ ] **Aplicar a migração** `c7d8e9f0a1b2` em cada ambiente (destrutiva) e
re-provisionar Observadores por projeto.

- [ ] Backfill/obrigatoriedade reforçada de `Connection.environment`
(o gate de produção depende dele estar correto).

- [ ] Evolução futura: gate por `EDITOR` específico nas ferramentas
stateless (exige escopo no payload + frontend).

## Mapeamento de strings legadas

O `AuthorizationService` aceita strings legadas e normaliza:

| Legado | Novo |
| --- | --- |
| `OWNER` (ws) | `ADMIN` |
| `MANAGER` | `ADMIN` |
| `CONSULTANT` | `EDITOR` |
| `GUEST` | `MEMBER` |
| `OPERATOR` | `VIEWER` |
| `OWNER` (client) | `ADMIN` |

Endpoints retornam **422** (não 500) quando o role enviado não cabe no
vocabulário aceito.
