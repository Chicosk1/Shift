# Lacunas

Ordenado por **impacto no caso de uso alvo**: automação de ajuste de preço a partir de pedidos
(`fluxos/vendas/job-pedido-ajuste-margem`).

Semeado na **Fase 0** (2026-08-20). A Fase 1 expande isto por lote; a Fase 2 fecha o que o MCP
conseguir confirmar.

---

## Bloqueadores do piloto

### L1 — Execução concorrente do mesmo fluxo agendado
**Impacto: alto.** O piloto roda a cada 5 minutos. Se uma execução demorar 6, o que acontece?
A doc de cron (`guias-de-uso/nos/agendamento-(cron).md` § "Limites e guardrails") **não trata
disso** — nada sobre `max_instances`, coalescing ou misfire do APScheduler. Só afirma que dois
workflows *distintos* no mesmo horário não conflitam, o que é outra pergunta.
**O que falta descobrir:** a plataforma serializa execuções do mesmo workflow, ou permite
sobreposição? Se permite, existe primitiva de trava?
**Como resolver:** pergunta para o suporte Viasoft. Não é inferível.

### L2 — Não existe primitiva de dry-run
**Impacto: alto.** O plano (§5.4) exige dry-run como modo padrão para o fluxo que escreve
preço. Não há recurso equivalente na plataforma. O toggle Teste/Produção limita a 500 linhas,
mas **não** suprime escrita — é limite de volume, não de efeito.
**O que falta descobrir:** confirmar que não há mesmo, e validar o padrão substituto (variável
booleana + `Filtro` desviando do nó de escrita para um nó de registro).

### L3 — Watermark e carga incremental
**Impacto: alto.** Termo com **zero ocorrências** nas 26 transcrições e nos 40 documentos.
O plano (§5.1) depende disso inteiramente.
**Hipótese a validar:** base de dados interna (teto de 200 mil linhas) como armazenamento do
watermark, lida/escrita por subfluxo `sub-obter-watermark`. Precisa confirmar leitura e escrita
na mesma execução sem condição de corrida.

### L4 — Margem: onde está cadastrada no ERP
**Impacto: alto.** O termo "margem" **não aparece em nenhuma fonte** — nem aula, nem doc. É
conhecimento exclusivo do ERP, não da plataforma.
**Agravante:** com o módulo 4 reposto, o acervo está completo — **380 KB de transcrição e
40 documentos — e o termo "margem" continua com zero ocorrências.** Não é lacuna de
material faltante; é conhecimento que só existe no seu ERP.

**O que falta descobrir (responder na Fase 5):** tabela e campo da margem; se é por item,
produto, categoria ou cliente; o que caracteriza "pedido novo"; qual campo de data usar como
marca d'água; se há histórico de preço.

### L5 — Teto de variação e disjuntor
**Impacto: alto.** §5.4 pede limite de variação percentual por item e limite de itens
alterados por execução. Nenhum dos dois existe como recurso.
**Hipótese a validar:** teto de variação via `Filtro` sobre coluna calculada no nó
`Matemática`; disjuntor via `Agregador` (COUNT) + `Filtro` que aborta. Falta descobrir **como
abortar um fluxo deliberadamente** — não foi visto nó de "falhar" ou "parar".

---

## Importantes, não bloqueadores

### L6 — Idempotência além do UPSERT
**Impacto: médio-alto.** Confirmado que existe `UPSERT` / "atualizar por chave" /
`insert_if_not_exists`. Não confirmado como evitar **oscilação** (o fluxo corrigir de novo o
que já corrigiu). O plano (§5.2) pede regra convergente.
Reforço independente de que isso importa: webhook em modo `immediately` é **at-least-once** e
pode executar duas vezes (`guias-de-uso/nos/webhook-(gatilho).md`).

### L7 — Nós de banco de dados não têm página de documentação
**Impacto: médio-alto.** `Introducao/conceitos.md` § "Nó" promete "o catálogo completo está na
seção Nós", e `guias-de-uso/nos/` **não contém** `SELECT`, `SQL Script`, `Inserção em Massa`,
`Limpar Tabela` nem `Destino SQL`. Para esses cinco a **aula é a única fonte**, o que os deixa
travados em `[UI-OBSERVADA]` — e são justamente os nós que o piloto usa para escrever.

### L8 — ~~Transcrições do módulo 4 estão vazias~~ RESOLVIDA (2026-08-20)
**Status: fechada.** Os 6 arquivos foram repostos: **2.122 linhas / 170 KB**, formato
consistente com o resto do acervo (593 marcações `(AÇÃO:)`, zero timestamps).

**Ressalva importante:** o módulo 4 é um caso de **migração de cadastro de itens** a partir do
banco de um concorrente (ConstruShow), com uso do copiloto para minerar o schema. **Não é o
caso de uso de preço/margem** — o termo "preço" aparece 1 vez e "margem" nenhuma. Ele fecha a
lacuna de "caso ponta a ponta com inserção real", mas **não** contribui para L4.

### L9 — Hierarquia divergente entre três documentos
**Impacto: médio.** `Introducao/conceitos.md` diz Organização → Espaço → Grupo econômico →
Projeto. `documentacao-tecnica/controle-de-acesso.md` diz Organização → Workspace → Projeto e
afirma que Cliente **não** é nível de acesso. `entidade-cliente.md` traz uma terceira variante.
**Leitura provável (a confirmar):** são hierarquias de propósitos diferentes — organização de
trabalho vs. permissão. Resolver no lote 2, antes que contamine o glossário.

### L10 — Como abortar/falhar um fluxo deliberadamente
**Impacto: médio.** Pré-requisito de L5. Existe menção a `dead_letter` como tipo de nó em
`exportar-e-importar.md` § "Cobertura V1" e a "Dead Letter" na categoria Banco de Dados em
`conceitos.md`, mas **nenhuma aula ou página de doc** explica o nó.

### L11 — Catálogo de tipos de variável de workflow
**Impacto: médio.** A aula (`m3-parte-1`) mostra 10 tipos na UI: Texto, Inteiro, Número,
Booleano, Objeto, Lista, Conexão, Arquivo, Segredo, Formulário. A doc só descreve `string`,
`connection` e `file_upload`. Sem catálogo confirmado, a parametrização do piloto fica em
`[UI-OBSERVADA]`.

### L12 — Consolidação de nós: quando
**Impacto: médio.** `consolidacao-de-nos-de-transformacao.md` documenta a fusão de 13 nós
legados em `transform`/`aggregate`/`combine`. **Sem data.** A base de conhecimento foi decidida
com o nome legado como chave primária e o sucessor anotado; se a migração chegar antes da
Fase 6, os arquivos de nó precisam de revisão.

### L13 — MCP não permite criar nem editar fluxo
**Impacto: médio.** As 15 ferramentas documentadas são 14 de leitura + `execute_workflow`.
Nenhuma de `create_*`/`update_*`. Se confirmado no dump real, a construção de fluxos é
**manual na interface**, e o papel do Claude Code passa a ser desenhar e revisar, não construir.
Isso muda a expectativa da Fase 5. Verificar assim que o MCP for registrado.

---

## Menores

### L14 — Relay é obrigatório para SQL Server on-premise?
Não documentado. O relay é agnóstico de banco (só TCP), mas as funções de **borda**
(extract/apply em Parquet) têm allowlist e dialeto **só Oracle** (`RELAY_*_dialect: oracle`,
e "Escopo Fase 1: só Oracle"). E `deploy-firebird.md` mostra acesso direto sem relay quando o
backend está na mesma rede.

### L15 — Sintaxe cron livre
A UI oferece frequências prontas (mínimo 5 min) e gera `cron_expression`. Não está documentado
se é possível **digitar** uma expressão arbitrária.

### L16 — Referência de API pública
Endpoints aparecem espalhados por vários documentos (`/workflows/{id}/execute`,
`/workflows/import`, `/workflows/{id}/clone`…), mas **não existe** página de referência,
OpenAPI, versionamento ou contrato estável.

### L17 — Seis documentos técnicos são proposta, não plataforma
`arquitetura-hibrida-(cloud-runner).md` ("Status: Proposta arquitetural"),
`upsert-rapido-staging-+-merge-set-based-(design).md`, `toleracia-a-erro-por-linha-na-borda-design.md`,
`extracao-na-borda-design.md`, `planejamento-dlt-(extracao-insercao).md` (flags OFF por
default, E2E não exercido) e `editor-de-documentacao-(keystatic).md` ("não funciona hoje").
Nunca marcar como `[CONFIRMADO-DOC]`.

### L18 — Migração de permissões pendente
`controle-de-acesso.md` registra a migração `c7d8e9f0a1b2` como **destrutiva e ainda não
aplicada**, exigindo re-provisionar Observadores. O modelo de papéis documentado pode não ser
o que está em produção hoje.
