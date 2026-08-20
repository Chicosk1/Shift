# Procedimento: conceder acesso

**Fontes:** `controle-de-acesso.md` `[CONFIRMADO-DOC]` (o desenho) e `m2p2:66-72`
`[UI-OBSERVADA]` (a prática).

> ⚠️ **Leia antes de aplicar.** `controle-de-acesso.md` declara status *"implementado"*, mas com
> uma **pendência operacional**: a migração `c7d8e9f0a1b2` é **destrutiva** (dropa
> `client_members`, remove convites `CLIENT`) e **não foi aplicada**, exigindo *"rollout coordenado
> e re-provisão de Observadores"*. **O modelo abaixo é o do código; o estado de cada ambiente pode
> ainda ser o antigo.** Ver `lacunas.md` L18.

---

## Escolha o nível antes de conceder

`[CONFIRMADO-DOC]` A hierarquia de **acesso** tem 3 níveis — o grupo econômico **não é degrau de
permissão**:

```
Organização  →  Workspace  →  Projeto
```

| Quem é a pessoa | Onde conceder | Papel |
|---|---|---|
| Consultor que vai trabalhar no espaço | **Workspace** | `VIEWER` / `EDITOR` / `ADMIN` |
| Consultor que administra a organização | **Organização** | `ADMIN` |
| **Cliente final**, só acompanhando | **Projeto** | `OBSERVADOR` (read-only) |
| Cliente final preenchendo formulário | — | **Link tokenizado** `/f/{token}`, sem papel |

> `[CONFIRMADO-DOC]` **Papel na organização não dá acesso a espaço.** Só o Org `OWNER` herda
> `ADMIN` em todos. `ADMIN` e `MEMBER` da org precisam de um `WorkspaceMember` **explícito** — que
> o próprio ADMIN pode se conceder. O motivo declarado é separar *"administrar a org"* de
> *"trabalhar no espaço"*. — `§ Regras de herança`

---

## A: conceder acesso a um espaço

`[UI-OBSERVADA]` Caminho da aula (`m2p2:70`): entre no espaço → **Configurações** → localize a
pessoa → conceda o acesso escolhendo o papel.
→ Em aula o papel escolhido foi **Administrador**, com a ressalva *"inicialmente"*.

`[CONFIRMADO-DOC]` No desenho atual, a tela é **Espaço › Acesso** — *"UMA tela: explícito +
herdado, ação inline, convite"*. Ela mostra o papel **efetivo** e a **origem**
(`explícito` / `herdado da org`), e permite mudar papel, remover e convidar ali mesmo.
— `§ Telas`

> ⚠️ `[CONFIRMADO-DOC]` Conceder `ADMIN` do workspace entrega, junto: chaves de API, webhooks,
> storage, deletar o workspace **e executar workflow em produção**. Para quem só vai construir
> fluxo, o papel correto é **`EDITOR`** — que já cria e edita fluxo, cliente, projeto e conexão, e
> executa em não-produção.

---

## B: convidar alguém que ainda não tem conta

`[CONFIRMADO-DOC]` A plataforma é **100% por convite** — não há signup público. Fluxo
"link-first": gere o convite e copie o `invite_url`. — `§ Convites`

1. Na tela de Acesso do nível desejado, use a **ação de convite** (não é mais tela separada).
2. Escolha o escopo:

| Escopo | Quem pode criar | O que é criado no aceite |
|---|---|---|
| `ORGANIZATION` | Org `OWNER`/`ADMIN` | `OrganizationMember` |
| `WORKSPACE` | Workspace `ADMIN` | `WorkspaceMember` |
| `PROJECT` | Workspace `EDITOR`+ | `ProjectObserver` (read-only) |

3. **E-mail é opcional** e funciona como restritor: informado, só quem tem aquele e-mail aceita
   (case-insensitive); em branco, **qualquer usuário logado aceita** — útil para link
   compartilhado, arriscado para o resto.
4. Copie o `invite_url` e envie.

`[CONFIRMADO-DOC]` **Convite multi-espaço:** um convite de escopo `ORGANIZATION` pode carregar
`workspace_grants` (`[{workspace_id, role}]`), criando no aceite o `OrganizationMember` **mais um
`WorkspaceMember` por grant**. Dá para definir no convite a quais espaços e com que papel.

`[CONFIRMADO-DOC]` Para **invalidar** um link: `POST /invitations/{id}/regenerate-link` gera novo
token e derruba o anterior, registrando `last_regenerated_at`. Não funciona em convite
`ACCEPTED`/`CANCELLED`.

**Limites:** 50 convites por usuário por hora; 50 `PENDING` por escopo.

---

## C: dar acesso de leitura ao cliente final

`[CONFIRMADO-DOC]` Vá ao **Projeto → aba "Observadores"** e provisione ou convide.
— `§ Telas`

O `OBSERVADOR` lê andamento, status e execuções **de um único projeto**. Não vê outros projetos,
não vê segredos, não edita. Para mais de uma pessoa ou mais de um projeto, **provisione cada
grant separadamente**.

`[VÍDEO]` `m2p2:66` mostra o efeito prático do isolamento: o segundo usuário tinha acesso ao
grupo econômico mas **nenhuma conexão aparecia**, porque as conexões estavam privadas.

---

## Verificar se ficou como você espera

`[VÍDEO]` A técnica usada em aula, e que vale copiar: **abra uma janela anônima e logue com o
outro usuário**, senão o login atual cai. — `m2p2:64`

`[CONFIRMADO-MCP]` Pelo MCP, sem trocar de usuário: **`list_project_members`** devolve os membros
de um projeto com papel e data de ingresso.

---

## Vocabulário: se aparecer um nome antigo

`[CONFIRMADO-DOC]` `§ Mapeamento de strings legadas` — o `AuthorizationService` aceita e
normaliza. `MANAGER` → `ADMIN`, `CONSULTANT` → `EDITOR`, `OPERATOR` → `VIEWER`,
`GUEST` → `MEMBER`, `OWNER` → `ADMIN`. Role fora do vocabulário devolve **`422`**.

É por isso que o MCP fala *"requer role MANAGER"* onde a matriz fala `ADMIN`.

---

## `[LACUNA]` Quem cria cliente e projeto — três fontes, duas respostas

- `controle-de-acesso.md § Matriz`: criar/editar/deletar cliente e projeto → **`EDITOR`**
- `entidade-cliente.md § Permissões`: criar/editar/deletar cliente → **`MANAGER`** (= `ADMIN`)
- MCP `create_project`: *"Requer role MANAGER no workspace"* (= `ADMIN`)

**Não escolhido.** Planejar com `ADMIN` é o caminho seguro. Ver L30.
