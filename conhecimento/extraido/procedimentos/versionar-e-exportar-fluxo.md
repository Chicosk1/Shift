# Procedimento: versionar, publicar e exportar um fluxo

**Fontes:** `modulo-3/modulo-3-parte-1.txt` (`m3p1:<linha>` — sem timestamp),
`guias-de-uso/exportar-e-importar.md`, `guias-de-uso/nos/agendamento-(cron).md`,
`guias-de-uso/nos/webhook-(gatilho).md`, `guias-de-uso/faq-perguntas-frequentes.md`, e o MCP.

Este arquivo fecha a `[LACUNA]` registrada no fim de `criar-fluxo.md` ("não foi demonstrado
publicar um fluxo nem alternar entre teste e produção").

---

## Parte 1 — Rascunho vs. versão publicada

### A regra, em uma linha

`[VÍDEO]` **Teste roda o rascunho. Produção roda a última versão publicada.**
Dito com essas palavras: *"Teste sempre vai considerar o que tá rascunho, produção sempre a versão
publicada"* — `m3p1:469`. Antecipado em `m3p1:406-408`: *"Quando você tá rodando em teste, ele vai
pegar sempre a versão de rascunho. […] Agora, se ele tiver em produção, ele sempre vai rodar
somente a versão última publicada."*

`[VÍDEO]` E de novo, sobre o Teste: *"o teste, ele roda o que tá agora. Não interessa se a versão
tá publicada ou não"* — `m3p1:440`.

### Onde isso aparece na tela

`[UI-OBSERVADA]` `m3p1:403-404`: no menu ⋯ há um **ícone de relógio** com o contador de alterações
pendentes — literalmente **"6 alterações desde a última versão (ver histórico)"**. O contador
sobe conforme você salva (`m3p1:423-424`: *"1 alteração desde a última versão"*; `m3p1:431`: duas)
e **volta a zero ao publicar** (`m3p1:417`: *"não tem mais nenhuma alteração"*).

`[VÍDEO]` A plataforma guarda o histórico sozinha: *"o sistema ele já tem uma inteligência que ele
vai salvando os históricos"* — `m3p1:405`.

`[UI-OBSERVADA]` Na **listagem de fluxos** existe uma coluna **VERSÃO**. Fluxo nunca publicado
mostra **`-`**. — `m3p1:410-411`

### Como publicar

1. Abra o menu **⋯**.
2. Clique em **Publicar versão**.
3. Confirme — **o número da versão é calculado pela plataforma**, você não digita.
   `[UI-OBSERVADA]` O botão vem rotulado com o número já pronto: **Publicar v1**, depois
   **Publicar v2**, depois **Publicar v3**. — `m3p1:415-418`, `m3p1:445-447`, `m3p1:462-463`
   `[VÍDEO]` *"ele já calcula automaticamente a próxima versão"* — `m3p1:417`.

### O que a aula demonstrou na prática

`[VÍDEO]` Duas provas de que Produção ignora o rascunho:

1. **Variável removida do rascunho continuou obrigatória em Produção.** A variável de conexão
   `ConConstruShow` foi apagada do painel de Variáveis; ao executar em **Produção**, a plataforma
   ainda barrou com **"Variável 'ConConstruShow' é obrigatória"**. O instrutor conclui: *"O
   sistema, ele tá na versão anterior, então ele tá ainda solicitando aquela variável em
   produção"*. Rodando em **Teste**, passou. — `m3p1:431-439`
   **Consequência importante: a declaração de variáveis é versionada junto com o fluxo.**

2. **Remapeamento de colunas só valeu depois de publicar.** Com as colunas `DEPARTAMENTO` e
   `DATA_ADMISSAO` alteradas apenas no rascunho, a execução em Produção gravou **NULL** nas duas.
   Depois de **Publicar v3**, a mesma execução (upsert) preencheu as colunas.
   — `m3p1:452-467`

> ⚠️ **Ressalva de fonte — a transcrição está duplicada nesse trecho.** Os blocos `m3p1:415-424` e
> `m3p1:444-451` são **quase idênticos palavra por palavra** (mesma frase *"eu vou rodar ele nulo.
> Vou dar como tá de novo"*, mesmo *"Agora eu vou voltar aqui […] pra adicionar departamento e
> tudo mais"*), e as narrações de `m3p1:427` (*"eu tirei as informações de departamento e data de
> admissão"*) e `m3p1:421` (*"vou voltar pra adicionar departamento"*) se contradizem.
> **A ordem exata da demonstração não é reconstituível** a partir deste arquivo. O **mecanismo**,
> porém, está afirmado em texto claro três vezes (`m3p1:406-408`, `m3p1:437`, `m3p1:469`) e
> corroborado por doc independente (abaixo) — esse eu registro como firme; a cronologia do
> exemplo, não.

### Corroboração independente do mecanismo

`[CONFIRMADO-DOC]` `guias-de-uso/nos/agendamento-(cron).md`: *"o agendamento só fica ativo quando
o workflow está em modo **Produção** e **Publicado**. Em modo de teste (draft), o nó pode ser
executado manualmente mas não dispara sozinho."*

`[CONFIRMADO-DOC]` `guias-de-uso/nos/webhook-(gatilho).md`: a **Production URL**
(`/api/v1/webhook/{path}`) *"só fica acessível após o workflow ser colocado em Produção e
publicado"*; a **Test URL** (`/api/v1/webhook-test/{path}`) roda o fluxo **em modo draft**.

`[CONFIRMADO-MCP]` `call_workflow` — *"Invoca outro workflow **publicado** como sub-rotina"* — e
tem campo **`version`**: *"Versão a usar: `'latest'` (padrão) ou número fixo"*.
**Isto é o pino de versão do subfluxo:** o fluxo pai pode travar numa versão publicada específica
em vez de seguir a mais recente.

`[CONFIRMADO-MCP]` `get_workflow` devolve *"se é template/publicado"*, e `list_workflows` devolve
o `status` — dá para auditar publicação por ferramenta, sem abrir a tela.

### Consequência prática — a cadeia inteira

`[INFERIDO]` Juntando as quatro fontes acima, publicar não é opcional: é o que **liga** a
automação. Sem publicar, um fluxo pronto **não é agendável** (cron não dispara), **não é
chamável por webhook de produção**, **não é usável como subfluxo** e **roda a versão errada** em
Produção. Para o piloto de ajuste de preço isso é o portão de entrada.

### Histórico de versões ≠ histórico de execuções

Duas coisas diferentes, que a tela mostra em lugares diferentes:

| | Onde | O que guarda | Fonte |
|---|---|---|---|
| **Histórico de versões** | Menu ⋯ → Histórico de versões (ícone de relógio) | As versões publicadas do desenho do fluxo | `[UI-OBSERVADA]` `m3p1:403`, `m3p1:473-474` |
| **Histórico de execuções** | Aba **Execuções** | Uma entrada por rodada, com status, duração e **snapshot imutável** do que rodou | `[CONFIRMADO-DOC]` `faq-perguntas-frequentes.md § Onde vejo o histórico de execuções` |

`[CONFIRMADO-DOC]` Sobre o snapshot: *"mesmo se você editar o fluxo depois, o histórico mostra
como **estava** quando rodou"*.

---

## Parte 2 — Exportar e importar

### ✅ L24 RESOLVIDA (neste lote) — não são versões concorrentes, são camadas diferentes

O conflito era: `m1:58` `[VÍDEO]` diz que exporta/importa *"através de JSON"*;
`exportar-e-importar.md` `[CONFIRMADO-DOC]` diz 4 formatos com round-trip **só em YAML**.

**A prova que faltava está no lote 4.** `[UI-OBSERVADA]` `m3p1:475-476`: no menu ⋯ o instrutor
aponta *"as opções de **Importar** e **Exportar** JSON/YAML"*. Ou seja: **JSON e YAML coexistem no
menu, hoje.** `m1:58` não estava errado sobre existir JSON — estava **incompleto**, e a frase
estava a serviço de outro assunto (o motivo de exportar: *"não existe transferência de fluxo entre
espaços"*), não de enumerar formatos.

**Conclusão:** vale a doc. Há mais de um formato, JSON canvas é um deles, e **o round-trip é
YAML**. `m1:58` fica registrado como imprecisão de aula, não como divergência de comportamento.

`[INFERIDO]` **A doc também se contradiz sozinha — e a contradição se resolve.** A abertura de
`exportar-e-importar.md` diz *"três formatos standalone"* e a tabela **Formatos suportados** lista
3 (SQL, Python, YAML); mas o passo 2 de **Como exportar pelo editor** diz *"Escolha um dos
**quatro** formatos"* e lista 4, incluindo `Exportar JSON (canvas)`. Leitura que reconcilia:
**3 formatos standalone + 1 formato de canvas = 4 itens de menu**. O JSON canvas não é
"standalone" (não roda fora do Shift), então ficou fora da tabela. Marcado `[INFERIDO]` porque a
doc não diz isso.

### Os formatos

`[CONFIRMADO-DOC]` `exportar-e-importar.md`:

| Item do menu | Extensão | Para que serve | Round-trip? |
|---|---|---|---|
| **Exportar JSON (canvas)** | `.json` | *"formato legado, restaura o canvas"* | `[LACUNA]` — ver L36 |
| **Exportar SQL (DuckDB)** | `.sql` | Auditar a lógica direto no DuckDB CLI | **Não** — somente leitura |
| **Exportar Python** | `.py` | Reproduzir o fluxo fora do Shift | **Não** — somente leitura |
| **Exportar YAML** | `.yaml` | **Versionar em git e re-importar em outro workspace** | **Sim** |

`[CONFIRMADO-DOC]` *"Os formatos SQL e Python são **somente leitura** — não há `import`
equivalente. Para round-trip, use **YAML**."*

`[CONFIRMADO-DOC]` O download começa na hora e o arquivo segue o slug do fluxo
(ex.: `order-enrichment.sql`).

### O que NÃO exporta para SQL/Python — e por que isso decide o assunto

`[CONFIRMADO-DOC]` Só **16 tipos de nó** são exportáveis para SQL e Python (V1):
`sql_database`, `inline_data`, `filter`, `mapper`, `record_id`, `sample`, `sort`, `join`,
`lookup`, `aggregator`, `deduplication`, `union`, `pivot`, `unpivot`, `text_to_rows`, `loadNode`.

Qualquer outro devolve **HTTP 422** com a lista de `node_id` e a razão; o editor abre um modal e
clicar num item seleciona o nó no canvas. As famílias recusadas:

- **Código arbitrário:** `code`
- **I/O externa:** `http_request`, `webhook`, `polling`, `csv_input`, `excel_input`, `api_input`,
  `extractNode`, `sql_script`
- **Efeitos colaterais:** `bulk_insert`, `composite_insert`, `truncate_table`, `notification`,
  `dead_letter`
- **Controle de fluxo:** `if_node`, `switch_node`, `loop`, `assert`, `call_workflow`, `manual`,
  `cron`
- **Aritmética legada:** `math`, `transformNode`

`[CONFIRMADO-DOC]` E **fluxo com ciclo não exporta em nenhum formato standalone** — *"o exportador
faz ordenação topológica e levanta erro se houver ciclo"*.

`[CONFIRMADO-DOC]` Duas limitações a mais do SQL exportado: o `pivot` não descobre os valores em
build-time (usa `PIVOT ... ON <col> USING SUM(...)` do DuckDB, que resolve em runtime, gerando
colunas `<valor>_<agg>`); e o `unpivot` exportado **exige `value_columns` explícito** —
`by_type` não é suportado.

### Variáveis e conexões no export standalone

`[CONFIRMADO-DOC]` Os exportadores **não pré-resolvem** `{{vars.X}}`:

- **SQL:** a variável vira placeholder shell-style **`${X}`**, declarada no cabeçalho com a lista
  de nós que a usam (`-- ${CUTOFF_DATE}   used in: extract_orders`).
- **Python:** vira `os.environ['CUTOFF_DATE']`.
- **Conexões** ficam como **TODO**: no SQL, um `ATTACH` comentado a substituir; no Python,
  `CONN_<id>_URL = os.environ.get(...)`.

Ou seja: o script exportado **não roda sem edição manual**. É artefato de auditoria, não de
deploy.

### Importar

`[CONFIRMADO-DOC]` Pela API:

```
POST /api/v1/workflows/import?workspace_id=<uuid>
Content-Type: multipart/form-data
file: <arquivo.yaml>
```

O backend **valida o `shift_version`** (rejeita major incompatível), extrai `nodes`, `edges` e
`variables`, e **cria um workflow draft** no workspace informado.
**O `workflow_id` original do YAML é descartado — sempre se gera um UUID novo.**

`[CONFIRMADO-DOC]` Pela tela: **⋯ → Importar YAML**. Depois do sucesso, o navegador é
redirecionado para o novo fluxo.

`[UI-OBSERVADA]` A aula mostra o item de menu como **"Importar"** ao lado de **"Exportar"**
JSON/YAML (`m3p1:476`), sem detalhar quais formatos o import aceita — ver L36.

### Estrutura do YAML — o que importa para editar à mão

`[CONFIRMADO-DOC]` O YAML tem `shift_version`, `workflow_id`, `workflow_name`, `exported_at`,
`settings` (com `variables`, `schedule`, `meta`), `nodes` e `edges`.

Dois detalhes que salvam tempo:

1. **`inputs` e `outputs` de cada nó são decorativos.** A doc é explícita: *"são derivados das
   `edges` apenas para legibilidade — o parser **ignora** esses campos ao reconstruir o workflow
   (única fonte de verdade são as `edges`)"*. Editar `inputs`/`outputs` no git **não faz nada**;
   a topologia mora em `edges`.
2. **`connection_id` sai como UUID literal** no exemplo da própria doc
   (`connection_id: 11111111-2222-3333-4444-555555555555`). Um YAML assim **não é portável** para
   outro workspace — o UUID não existe lá. Para portabilidade, o fluxo precisa referenciar a
   conexão por variável antes de exportar: ver `parametrizar-com-variaveis.md`.

`[CONFIRMADO-DOC]` `settings.schedule` existe no esquema (aparece como `schedule: null` no
exemplo). **Isso é a diferença central:** o cron **sobrevive ao YAML**, e é justamente um dos
nós que o export SQL/Python recusa com 422.

---

## Parte 3 — Recomendação: qual formato usar no repositório git

### **Use YAML. Não é preferência de estilo — os outros três não servem.**

**1. É o único que volta.** `[CONFIRMADO-DOC]` SQL e Python são *"somente leitura — não há
`import` equivalente"*. Versionar num formato que não reimporta é guardar documentação, não código.
E a própria doc nomeia o caso de uso na tabela de formatos: YAML = *"Versionar em git e
re-importar em outro workspace"*.

**2. Para uma automação de ajuste de preço, SQL/Python não é pior — é impossível.**
`[CONFIRMADO-DOC]` `bulk_insert` e `cron` estão **os dois** na lista de 422. Um fluxo que ajusta
preço **precisa escrever** (`bulk_insert`) e, se for automático, **precisa agendar** (`cron`).
Somando o resto do desenho previsto — `if_node` com `decision_mode: single` para o disjuntor
(`lacunas.md` L2/L5), `sql_database` para ler pedidos (alias `extractNode`, **também na lista de
422**) e `math` para a coluna calculada — a exportação SQL/Python devolveria 422 apontando
praticamente todos os nós do fluxo. **Não há trade-off a avaliar.**

**3. YAML preserva o que o piloto precisa preservar:** `settings.variables` (com `name`, `type`,
`required`) e `settings.schedule`. O JSON canvas `[LACUNA]` não tem estrutura documentada em
nenhuma fonte deste acervo — não se sabe o que ele guarda.

**4. YAML tem verificação de compatibilidade.** `[CONFIRMADO-DOC]` O import valida
`shift_version` e **rejeita major incompatível**, em vez de importar torto. É o único formato com
contrato de versão declarado.

**5. YAML é diffável.** `[INFERIDO]` Texto, uma chave por linha, ordem estável dentro de `nodes`
e `edges`. Um `git diff` mostra "mudou a query do nó `extract_orders`". Diff de JSON canvas com
coordenadas de posição é ruído; diff de `.sql` gerado não diz o que mudou no desenho.

### Duas ressalvas que precisam entrar no processo, não só na escolha

⚠️ **Importar é criar, não atualizar.** `[CONFIRMADO-DOC]` O import *"cria um workflow **draft**"*
e **descarta o `workflow_id`**, gerando UUID novo. Não há, em nenhuma fonte deste acervo,
endpoint de "atualizar fluxo existente a partir de YAML". **Consequência para o git:** o fluxo
`git → Shift` **não é deploy, é clone**. Cada import gera um fluxo novo, com histórico de versões
zerado e `VERSÃO = -`. Um pipeline de CI que reimporte a cada commit acumula duplicatas. Ver L37.

⚠️ **Parametrize a conexão antes de exportar.** Sem isso, o YAML carrega o UUID de conexão do
workspace de origem e o fluxo importado aponta para o vazio. Ver
`parametrizar-com-variaveis.md § Variável de Conexão`.

### Uso legítimo dos outros três

- **JSON canvas** — restaurar o canvas no mesmo ambiente. É declarado **legado**; não use como
  formato de repositório.
- **SQL (DuckDB)** — `[CONFIRMADO-DOC]` *"Auditar a lógica do fluxo direto no DuckDB CLI"*. Bom
  para revisar a transformação de um trecho **só de leitura** (ex.: o cálculo de preço sugerido
  isolado, antes do `bulk_insert`). Útil em revisão, inútil em versionamento.
- **Python** — reproduzir fora do Shift para investigação.

---

## Lacunas deste procedimento

`[LACUNA]` **L36 — o "Importar" da tela aceita JSON, ou só YAML?** `exportar-e-importar.md`
`[CONFIRMADO-DOC]` só documenta import de YAML (`⋯ → Importar YAML`, e o endpoint recebe
`arquivo.yaml`). `m3p1:476` `[UI-OBSERVADA]` mostra o par "Importar"/"Exportar" rotulado
**JSON/YAML**, sem separar. Se o JSON canvas **também** reimporta, ele volta a ser candidato a
formato de repositório e a recomendação acima precisa ser reavaliada.
**Impacto: médio.** **Falta descobrir:** os itens literais do menu ⋯ na versão atual, e se existe
`POST .../import` aceitando `.json`.

`[LACUNA]` **L37 — como atualizar um fluxo existente a partir de YAML.** Import documentado só
cria draft com UUID novo. Sem isso, não existe pipeline `git → ambiente` idempotente.
**Impacto: alto** para operar o piloto por repositório; **baixo** se o git for só registro
histórico. **Falta descobrir:** se há endpoint de update/replace por `workflow_id`, ou se o
caminho oficial é importar e depois apagar o anterior à mão.

`[LACUNA]` **L38 — o export pega o rascunho ou a versão publicada?** Isso decide o significado de
todo commit. `exportar-e-importar.md` não menciona versão em lugar nenhum, e o YAML não tem campo
de versão publicada (tem `exported_at`, não `version`). Como Teste roda rascunho e Produção roda a
publicada (`m3p1:469`), exportar sem saber qual das duas está no arquivo torna o repositório
ambíguo. **Impacto: alto.** **Falta descobrir:** se há escolha na hora de exportar, ou qual é o
padrão.

`[LACUNA]` **L39 — Produção sem nenhuma versão publicada: o que roda?** `m3p1:409` diz *"é capaz
de até dar erro, porque eu nunca publiquei nenhuma versão"*; `m3p1:414`, depois de testar, diz
*"quando não tem uma publicação ele vai pegar sempre a atual"* — ou seja, cai no rascunho. A fala
é hesitante (*"É, até deve... é..."*), o que a torna registro fraco, e nenhuma doc confirma.
**Impacto: médio** — um fluxo em Produção nunca publicado se comportaria como Teste sem limite de
500 linhas. **Falta descobrir:** o comportamento real.

`[LACUNA]` **L40 — `version: 'latest'` do `call_workflow` é a última publicada ou o rascunho?**
`[CONFIRMADO-MCP]` o campo aceita `'latest'` (padrão) ou número fixo, e a descrição diz *"workflow
publicado"*. Se `latest` significar rascunho, um subfluxo em edição afeta o pai em Produção.
**Impacto: médio-alto** para o padrão `<ENTIDADE> - CORE` da Viasoft (`m3p1:295-316`), que é todo
baseado em subfluxo. **Falta descobrir:** a semântica exata de `latest`.

`[LACUNA]` **L41 — dá para voltar a uma versão anterior?** O menu ⋯ tem **Histórico de versões**
(`m3p1:473-474`), mas a aula só diz *"que a gente já falou"* e não abre o painel. Não se sabe se
há rollback, comparação entre versões, ou anotação de release.
**Impacto: médio** para operação de produção. **Falta descobrir:** o conteúdo do painel.

`[LACUNA]` **L42 — a doc de export cita tipos de nó que o MCP não confirma.** A lista de cobertura
V1 nomeia `loadNode`, `transformNode`, `composite_insert`, `switch_node`, `assert`, `polling`,
`api_input`, `excel_input`. Alguns são aliases conhecidos (`extractNode` → `sql_database`
`[CONFIRMADO-MCP]`), outros não foram verificados. Reforça `lacunas.md` L23.
**Impacto: baixo.** **Falta descobrir:** o mapa alias→tipo canônico completo.
