# Glossário da plataforma Shift

Lote 1 da Fase 1 (2026-08-20). Fontes: `Introducao/{visao-geral,conceitos,primeiros-passos}.md`,
`transcricoes/modulo-1/modulo-1.txt` (citado como `m1:<linha>` — a transcrição **não tem
timestamp**), e o dump do MCP.

**Coluna "MCP/API"** = como o mesmo conceito aparece na superfície técnica. Divergência de nome
entre interface e API é a causa nº 1 de erro ao configurar nó por ferramenta.

---

## Hierarquia

### Organização
`[CONFIRMADO-DOC]` A empresa dona da instância do Shift. Teto do isolamento: dados de uma
empresa nunca encostam nos de outra. — `conceitos.md § Organização`

`[VÍDEO]` Na prática existe **uma só** (Viasoft) e não muda. Visível no rodapé do menu lateral;
se houvesse mais de uma, seria possível trocar ali. — `m1:44`

| Interface | MCP/API |
|---|---|
| Organização | `organization` (escopo de permissão) |

### Espaço (Workspace)
`[CONFIRMADO-DOC]` "Um produto ou time dentro da organização". É a **unidade de colaboração**.
Trocar de espaço troca o conjunto de clientes, conexões e modelos visíveis. — `conceitos.md § Espaço`

`[VÍDEO]` Na Viasoft o critério real é **vertical de ERP**, não time: os espaços observados são
`ConstruShow`, `Agrotitan`, `Siagri`, `PetroShow`, `Talent`. — `m1:44-45`

`[UI-OBSERVADA]` A interface abrevia como **"WS"** no seletor de contexto (ex.: "WS
ConstruShow"). Propriedades editáveis pelo ícone de lápis: sistemas do espaço (incluindo
**concorrentes**, cada um com seu banco), cor, ícone (predefinido ou upload), nome, e a
organização a que pertence (somente leitura). Uma **estrela** define o espaço favorito, que
passa a ser o de login. Sem favorito, entra no primeiro. — `m1:46-50`

`[VÍDEO]` Criar espaço é **exclusivo de administrador**. — `m1:47`

> ⚠️ **`[LACUNA]`** `conceitos.md § Espaço` tem uma **frase truncada** no original: *"é nele que
> vivem"* — e a lista do que vive num espaço **não existe** no documento. Falta descobrir a
> enumeração canônica. Pelas aulas, ao menos: conexões, bases de dados internas, arquivos,
> modelos de entrada, formulários, fluxos-template, nós personalizados, grupos econômicos.

| Interface | MCP/API |
|---|---|
| Espaço, WS | `workspace` / `workspace_id` |

### Grupo econômico (Cliente)
`[CONFIRMADO-DOC]` O cliente contratante. Chama-se "grupo econômico" porque agrupa **vários
CNPJs** — matriz (marcada como principal) e filiais. Carrega **Fase** (`prospect` → `onboarding`
→ `go_live` → `live` → `maintenance` → `offboarding`) e **Criticidade** (`low`, `medium`, `high`,
`critical`). — `conceitos.md § Grupo econômico (Cliente)`

`[CONFIRMADO-DOC]` Distinção explícita: *"o grupo econômico é com quem você trabalha; o projeto é
o que você faz para ele"*. Antes os dois se confundiam. — `conceitos.md § Grupo econômico`

`[VÍDEO]` É o cliente **do Shift**, não necessariamente o cliente da Viasoft. O agrupamento de
CNPJs existe para validação de licença. — `m1:41`

`[UI-OBSERVADA]` É o único item de menu que fica **sempre em evidência**, por fazer parte da
hierarquia. Dentro dele: estabelecimentos, arquivos, formulários, conexões, janelas, histórico,
e **Projetos**. — `m1:51-53`

> ⚠️ **CONFLITO (D1 / L9)** — `documentacao-tecnica/controle-de-acesso.md § Hierarquia` afirma
> que a hierarquia de **acesso** tem 3 níveis (Organização > Workspace > Projeto) e que *"Cliente
> continua existindo como entidade, mas não tem mais membership de acesso"*. Já `conceitos.md`
> e `m1` apresentam 4 níveis com o Grupo econômico dentro. **Não escolhi**: a leitura provável é
> que sejam hierarquias de propósitos diferentes (organização de dados vs. permissão), mas
> **falta confirmar**.

| Interface | MCP/API |
|---|---|
| Grupo econômico, Cliente | `client` — na URL aparece como `/c/{grupo}` |

### Projeto
`[CONFIRMADO-DOC]` Um engajamento específico dentro de um grupo econômico — tipicamente uma
migração ou integração contratada. É a "pasta de trabalho" onde os fluxos moram e rodam.
Pode ser marcado como **restrito**, visível só para quem foi adicionado. O cliente final pode
receber acesso de **leitura** a um projeto, sem entrar no espaço. — `conceitos.md § Projeto`

`[VÍDEO]` O motivo real de existir o nível: equipes distintas (implantação, produto,
personalizações) tocam o mesmo cliente sem ver o trabalho uma da outra, e o acesso é concedido
por projeto. — `m1:43`

`[CONFIRMADO-MCP]` `list_project_members` devolve papéis **`EDITOR` ou `CLIENT`**.

| Interface | MCP/API |
|---|---|
| Projeto | `project` / `project_id` |

### A hierarquia completa
`[CONFIRMADO-DOC]` — `conceitos.md § A hierarquia`

```
Organização              a empresa dona da instância
└── Espaço               um produto ou time (Workspace)
    └── Grupo econômico  o cliente contratante; agrupa os CNPJs
        └── Projeto      um contrato/engajamento
            └── Fluxo    o trabalho de dados em si
```

Cada nível **contém vários** do de baixo. Existe por três motivos declarados: **isolamento**
(dados de uma empresa nunca encostam nos de outra), **reuso** (o que se define num nível fica
disponível abaixo) e **permissão** (o papel num nível vale para tudo que ele contém).

`[CONFIRMADO-DOC]` Aparece na URL: `…/w/{espaço}/c/{grupo econômico}/projetos`, e no rodapé do
menu lateral. — `conceitos.md § A hierarquia`

`[UI-OBSERVADA]` Navegação por **breadcrumb** no topo permite subir de projeto → grupo → espaço.
— `m1:54-55`

---

## As três peças do dia a dia

### Conexão
`[CONFIRMADO-DOC]` Apontamento para banco externo (Oracle, SQL Server, Postgres, Firebird,
MySQL). Credenciais cadastradas **uma vez** e reusadas em N fluxos — equivale a um DSN /
connection string guardado. Senhas **nunca trafegam em texto claro de volta** para o usuário.
— `conceitos.md § Conexão`

`[CONFIRMADO-MCP]` `get_connection` devolve nome, tipo, host, porta, banco e usuário, e declara
explicitamente: *"Nunca retorna senhas, tokens ou strings de conexão"*. **Verificado na prática
em 2 conexões Oracle reais** (Fase 2) — nenhuma credencial retornada.

`[CONFIRMADO-MCP]` **Conexão Oracle funciona sem relay:** `test_connection` retornou SUCESSO em
duas conexões Oracle diretas (host `192.168.90.218`). Ver `divergencias.md §0`.

**Escopo** `[CONFIRMADO-DOC]` — toda conexão pertence a **exatamente um** dono:

| Escopo | Quem enxerga | Usar quando |
|---|---|---|
| Espaço (compartilhada) | todos os clientes e projetos do espaço | banco interno comum, sandbox |
| Grupo econômico | todos os projetos daquele cliente | **o caso mais comum** — o banco é do cliente |
| Projeto | apenas aquele projeto | isolamento máximo |

Regra de bolso declarada: na dúvida, **grupo econômico**, porque o banco quase sempre é do
cliente. — `conceitos.md § Conexão`

> ⚠️ **CONFLITO (D10)** — o MCP não expõe esse escopo de 3 valores. `get_connection` devolve
> **`Publica: Sim/Não`** e **`Workspace: <uuid>`**, sem nenhum campo de grupo econômico ou
> projeto. `[LACUNA]` Falta saber se `Publica` + ausência de `client_id`/`project_id` **é** a
> representação do escopo "Espaço", ou se o MCP só não expõe os outros dois.

`[VÍDEO]` Credencial sozinha não faz nada — "ganha vida dentro de um fluxo". — `m1:38`

| Interface | MCP/API |
|---|---|
| Conexão | `connection` / `connection_id` (UUID) |

### Fluxo
`[CONFIRMADO-DOC]` Diagrama de passos conectados (os nós) que extrai, transforma e carrega.
Análogo declarado: um `WITH` encadeado em SQL, onde cada CTE é um nó. — `conceitos.md § Fluxo`

Existe em **duas formas** `[CONFIRMADO-DOC]`:

- **Instância** — vive num **projeto** e roda contra os dados daquele cliente.
- **Template** — vive no **espaço**, sem cliente, e serve de molde. Você **clona** para criar
  instância em cada projeto.

`[CONFIRMADO-DOC]` Clonar não é a única forma: um projeto pode **usar** um fluxo do espaço
diretamente como sub-fluxo, sem cópia — útil quando a correção no template deve valer para todos
os usuários de uma vez. O dono controla quem enxerga e quem clona; é possível **liberar o uso sem
liberar a cópia** (ex.: nó personalizado cuja lógica interna fica oculta).
— `conceitos.md § Fluxo`

`[UI-OBSERVADA]` Na listagem: organização por **pastas** (arrastar o fluxo para dentro),
**tags**, filtros rápidos "em teste" / "em produção", busca textual, e troca de visualização.
Menu por item: abrir, ver documentação, exportar, duplicar, acesso e visibilidade, mover para
pasta. — `m1:56-58`

`[UI-OBSERVADA]` Ao criar: **em branco** ou **a partir de um template**; nome; escolha entre
"só eu vejo" e "compartilhar com o time" (que significa **todos do espaço**); em Mais Opções,
descrição e tag. — `m1:59-60`

| Interface | MCP/API |
|---|---|
| Fluxo | `workflow` / `workflow_id` (UUID) |

### Nó
`[CONFIRMADO-DOC]` Cada caixinha do fluxo. Conecta-se arrastando da **bolinha de saída** até a
**bolinha de entrada** do próximo. — `conceitos.md § Nó`

`[VÍDEO]` Regra de forma: todo nó tem uma entrada e uma saída, **exceto os de gatilho**, que só
têm saída. — `m1:39`, confirmado na prática em `m1:65`

`[CONFIRMADO-MCP]` Cada tipo de nó tem **nível de risco** declarado e pode ter "armadilhas
conhecidas" no próprio contrato — visíveis via `describe_node`. São **três** níveis observados:

| Risco | Significado | Exemplos |
|---|---|---|
| `read_only` | Não escreve | `manual`, `cron`, `csv_input`, `mapper`, `math`, `filter`, `if_node`, `sql_database` |
| `write` | Escreve | `bulk_insert`, `internal_data_write` |
| **`unknown_write`** | A plataforma **não sabe estaticamente** se escreve | `sql_script` (SQL arbitrário) |

`unknown_write` não aparece em nenhuma documentação — ver `divergencias.md` D11.

| Interface | MCP/API |
|---|---|
| Nó, caixinha | `node` / `node_id`; o tipo é `node_type` (ex.: `bulk_insert`) |
| Ligação entre nós | `edge` / `edge_id`, com `sourceHandle` e `targetHandle` |

---

## Categorias de nó

`[CONFIRMADO-DOC]` `conceitos.md § Nó` lista: **Gatilhos**, **Entradas**, **Transformação**,
**Estatística**, **Banco de Dados**, **Saídas**, **Integrações**, **Lógica**, **IA**, **Outros**.

`[CONFIRMADO-MCP]` `list_nodes` expõe 9 categorias: `trigger`, `input`, `transform`,
`statistics`, `logic`, `database`, `output`, `subflow`, `other`.

> ⚠️ **CONFLITO** — três diferenças, registradas sem escolha:
>
> 1. **"Integrações" não é categoria no MCP.** A descrição de `list_nodes` diz literalmente:
>    *"Integrações não são uma categoria: procure pelo NOME do serviço"*. Google Sheets e Drive
>    aparecem como `input`/`output`. Mas `conceitos.md` a apresenta como categoria.
> 2. **`subflow` existe no MCP e não em `conceitos.md`**, que trata Entrada/Saída de fluxo como
>    Gatilho e Saída.
> 3. **`conceitos.md` cita nós que `list_nodes` não devolve:** "Monitorar Mudanças" (gatilho),
>    "Dead Letter" (banco), "Gmail (enviar e-mail)", "WhatsApp via Z-API", "Analista IA",
>    "Decisão IA", "Nota", "Grupo". `visao-geral.md` reforça e-mail e WhatsApp. `m1:65` cita
>    "Monitoramento" na biblioteca de gatilhos `[UI-OBSERVADA]`.
>
> **`[LACUNA]`** Falta descobrir se esses nós (a) não existem mais, (b) existem e `list_nodes`
> os omite, ou (c) estão restritos pela whitelist `allowed_tools` da API Key. **Nota:** "Nota" e
> "Grupo" são elementos de organização do canvas e provavelmente não são nós de engine.

---

## Termos de execução e ciclo de vida

| Termo | Definição | Fonte |
|---|---|---|
| **Canvas** | A tela onde os nós moram | `[CONFIRMADO-DOC]` `primeiros-passos.md §1` |
| **Biblioteca de nós** | Painel lateral com o catálogo. Atalho **`L`** abre e fecha | `[UI-OBSERVADA]` `m1:64` |
| **Preview** | Prévia por nó, sem gravar. Executar dentro da configuração de um nó roda **os nós anteriores até ele** | `[CONFIRMADO-DOC]` `primeiros-passos.md §2`; `[VÍDEO]` `m1:68` |
| **`row_count`** | Quantas linhas foram lidas e gravadas; aparece no painel de execução | `[CONFIRMADO-DOC]` `primeiros-passos.md §6` |
| **Logs** | Painel inferior: por passo, o que aconteceu e o tempo de execução. Resultado visualizável em tabela | `[UI-OBSERVADA]` `m1:69` |
| **Execução** | Uma rodada do fluxo. Estados `RUNNING`/`COMPLETED`/`FAILED`/`CANCELLED`/`CRASHED` | `[CONFIRMADO-MCP]` `get_execution_status` |
| **Plano de execução** | Item de menu sob Automação, ao lado de Fluxos e Nós personalizados | `[UI-OBSERVADA]` `m1:45` |
| **Modelo de entrada** | Contrato de colunas que valida o arquivo **antes** de processar, falhando com mensagem clara | `[CONFIRMADO-DOC]` `primeiros-passos.md §2` |
| **Nó personalizado** | Item de menu sob Automação. Ligado ao "liberar uso sem liberar cópia" de `conceitos.md` | `[UI-OBSERVADA]` `m1:45` |
| **`shift-upload://<file_id>`** | Esquema de URI para arquivo enviado à plataforma | `[CONFIRMADO-MCP]` config do `csv_input` |
| **DuckDB** | Motor de dados sob os nós. Materializa datasets e executa as expressões SQL | `[CONFIRMADO-MCP]` descrições de `csv_input` e `mapper` |

---

## O que o Shift é e não é

`[CONFIRMADO-DOC]` **Use quando** o trabalho é mover e transformar dados entre origens e destinos
de forma repetível, auditável, sem depender de alguém rodar script na mão.

**Provavelmente não é o Shift quando** você precisa de aplicação interativa com regra de negócio
por requisição (isso é backend de produto) ou de BI para explorar dados visualmente. *"O Shift
alimenta esses sistemas; ele não os substitui."* — `visao-geral.md § Quando usar (e quando não)`

`[VÍDEO]` Posicionamento dado em aula: nasceu para **migração de dados** de concorrentes, e
depois ganhou integração e automação. Comparações explícitas — **n8n** trabalha com dados em
memória e não escala para milhões de linhas; **Pentaho** é dos anos 2000, tem instalação
complexa (pacotes Java) e não recebe mais atualização; **Airflow** é orquestrador de código
Python e exige programador. O Shift é **low code, não no code**. — `m1:6-10`

`[VÍDEO]` A plataforma é **100% por convite** — não há autocadastro. À data da gravação, só
e-mails do domínio Viasoft. Login por e-mail/senha ou conta Google. — `m1:35-36`
Coerente com `controle-de-acesso.md § Convites` `[CONFIRMADO-DOC]`.
