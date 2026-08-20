---
title: Entidade Cliente
---

## Hierarquia de dados

```
Organization
  └── Workspace
        └── Client          ← NOVO
              ├── ClientCnpj[]   (Grupo Econômico)
              └── Project[]
                    └── Workflow[]
```

**Connection** continua como recurso transversal com exatamente UM proprietário:

- `workspace_id` — conexão compartilhada do workspace

- `project_id`   — conexão exclusiva de projeto (legado)

- `client_id`    — conexão do cliente (novo)

---

## Modelo Cliente

| Campo | Tipo | Default |
| --- | --- | --- |
| `id` | UUID | gerado |
| `workspace_id` | UUID FK → workspaces | obrigatório |
| `name` | varchar(255) | obrigatório |
| `legal_name` | varchar(255) | opcional |
| `phase` | enum: prospect / onboarding / **go_live** / live / maintenance / offboarding | `onboarding` |
| `criticality` | enum: low / **medium** / high / critical | `medium` |
| `notes` | text | opcional |
| `created_by_id` | UUID FK → users | opcional |

## Grupo Econômico (client_cnpjs)

Cada Cliente pode ter N CNPJs (CNPJ matriz + filiais). Exatamente um deve
ser marcado como `is_primary = true`.

Validação: formato 14 dígitos + verificação dos dígitos verificadores (mod 11).

---

## Migration

```
# Aplicar
alembic upgrade head

# Rollback (desenvolvimento apenas — perda parcial de dados possível)
alembic downgrade -1
```

### O que a migration faz

1. Cria tabelas `clients` e `client_cnpjs`

2. Adiciona `client_id` em `projects` (nullable temporariamente)

3. Adiciona `client_id` em `connections` (nullable)

4. Remove as 2 CheckConstraints antigas da tabela `connections`

5. **Data migration**: cria 1 `Client` por `Project` existente (phase=`live`, criticality=`medium`)

6. Vincula cada projeto ao seu cliente

7. Migra conexões `project_id` → `client_id` e nula `project_id`

8. Torna `projects.client_id` NOT NULL

9. Adiciona nova CheckConstraint: exatamente um dos três campos deve ser NOT NULL

### Aviso sobre downgrade

O downgrade é aproximado: conexões migradas retornam ao *primeiro projeto do
cliente* (ORDER BY created_at LIMIT 1). Em ambiente de produção com dados
divergidos, o rollback correto é restaurar um backup.

---

## Endpoints novos

```
GET    /api/v1/workspaces/{ws_id}/clients
POST   /api/v1/workspaces/{ws_id}/clients        (requer role >= MANAGER)

GET    /api/v1/clients/{client_id}
PUT    /api/v1/clients/{client_id}               (requer role >= MANAGER)
DELETE /api/v1/clients/{client_id}               (requer role >= MANAGER)

GET    /api/v1/clients/{client_id}/cnpjs
POST   /api/v1/clients/{client_id}/cnpjs         (requer role >= MANAGER)
DELETE /api/v1/clients/{client_id}/cnpjs/{id}    (requer role >= MANAGER)

GET    /api/v1/clients/{client_id}/projects
GET    /api/v1/clients/{client_id}/connections
```

## Endpoints alterados

| Endpoint | Mudança |
| --- | --- |
| `POST /api/v1/workspaces/{ws_id}/projects` | **Breaking**: body agora exige `client_id` |
| `GET /api/v1/projects/{id}/connections` | Passa a incluir também conexões do cliente |
| `POST /api/v1/connections` | Aceita `client_id` como terceira opção de owner |

## Permissões

Clientes herdam o modelo de permissão do Workspace:

| Operação | Role mínimo |
| --- | --- |
| Listar / ler cliente ou CNPJs | `VIEWER` |
| Criar / editar / deletar cliente | `MANAGER` |
| Criar / deletar CNPJs | `MANAGER` |

Não existe `ClientMember` nesta fase — membros do workspace têm acesso
proporcional ao seu papel no workspace.
