# Procedimento: usar o editor de fluxo (a tela)

**Fonte principal:** `modulo-3/modulo-3-parte-1.txt` — citada como `m3p1:<linha>`. A transcrição
**não tem timestamp**; a linha é a unidade de citação. É o tour guiado do editor
(`m3p1:7` declara o objetivo: *"dar uma olhada em todos os recursos da tela"*).
**Complementos:** `guias-de-uso/faq-perguntas-frequentes.md`, e o MCP para os contratos.

**Pré-requisito:** ter um fluxo criado — ver `criar-fluxo.md`. O canvas de um fluxo novo
`[VÍDEO]` **começa sempre vazio** (`m3p1:28`).

---

## 1. A barra de ferramentas do canvas (lateral superior esquerda)

`[UI-OBSERVADA]` `m3p1:30` — *"opções de atalho e funcionalidades de manipulação da tela"*.

| Ferramenta | Atalho | O que faz | Fonte |
|---|---|---|---|
| **Mão** (mãozinha) | **`H`** | Move/arrasta o canvas | `[UI-OBSERVADA]` `m3p1:33-36` |
| **Seta** (mouse/seleção) | `[LACUNA]` — não narrado | Seleção; permite marcar vários nós de uma vez e apagar com `Delete` | `[UI-OBSERVADA]` `m3p1:37-39` |
| **Biblioteca de nós** | **`L`** | Abre e fecha o catálogo de nós | `[UI-OBSERVADA]` `m3p1:40-41` |
| **Organizar nós em grupos** | `[LACUNA]` | Abre painel à direita listando nós e o grupo de cada um; permite mover nó para um grupo | `[UI-OBSERVADA]` `m3p1:82-84` |
| **Agrupar** | `[LACUNA]` | Agrupa os nós selecionados | `[UI-OBSERVADA]` `m3p1:76-77` |
| **Adicionar nota** | **`N`** | Cria bloco de notas no canvas | `[UI-OBSERVADA]` `m3p1:160-161` |
| **Tela cheia** | **`F`** | Esconde os menus laterais; sobram canvas, **Salvar** e **Executar** | `[UI-OBSERVADA]` `m3p1:182-189` |
| **Desfazer / Refazer** | **`Ctrl+Z`** / **`Ctrl+Shift+Z`** | Desfaz e refaz (demonstrado com exclusão de ligação) | `[UI-OBSERVADA]` `m3p1:175-181` |
| **Zoom out / Zoom in** | scroll do mouse | Botões `-` e `+`. **Scroll puro já dá zoom — não precisa de `Ctrl`** | `[UI-OBSERVADA]` `m3p1:190-193` |
| **Ajustar tela** | `[LACUNA]` | Reenquadra: volta para onde estão os nós e tenta caber o fluxo inteiro na tela | `[UI-OBSERVADA]` `m3p1:195-198` |

`[UI-OBSERVADA]` **Ancoramento/alinhamento:** ao arrastar um nó, quando as extremidades coincidem
com as de outro, a tela exibe guias de alinhamento. É só apoio visual. — `m3p1:68-71`

### Arrastar vs. Seleção — a diferença completa

`[CONFIRMADO-DOC]` `faq-perguntas-frequentes.md § "Modo arrastar" vs "Modo seleção"`:

- **Arrastar** (mãozinha): clicar e arrastar **move o canvas**. Selecionar nó exige **um clique**
  no nó.
- **Seleção** (seta): clicar e arrastar **seleciona múltiplos nós**. Para mover o canvas,
  **espaço + arrastar**.
- Default sugerido para quem está aprendendo: **Arrastar**.

`[VÍDEO]` O instrutor alterna entre as duas pelo teclado durante todo o trabalho e recomenda o
hábito: *"concorrentemente isso facilita bastante pra trabalhar"* (`m3p1:73-75`). Só o atalho da
mão (`H`) foi dito em voz alta; o da seta ficou `[LACUNA]`.

---

## 2. Biblioteca de nós (`L`)

`[UI-OBSERVADA]` `m3p1:40-47`:

1. Abra com **`L`** (ou pelo ícone).
2. Alterne a visualização entre **cards** e **lista**.
3. Use o **campo de busca** para achar nó por nome.
4. Ou role pelos **grupos**.
5. **Arraste** o nó da biblioteca para o canvas.

`[UI-OBSERVADA]` Os grupos citados literalmente na aula: **gatilho, entrada, transformação de
dados, lógica, banco de dados, saídas, IA e outros** (`m3p1:44`).

> ⚠️ **CONFLITO já registrado — não é novo.** Essa lista de 8 não bate com as 10 categorias de
> `conceitos.md` nem com as 9 de `list_nodes` `[CONFIRMADO-MCP]`. Duas diferenças que a aula
> acrescenta: ela **tem "IA"** (que o MCP não tem como categoria) e **não tem "Estatística"**
> (que o MCP tem, com 8 análises prontas). O conflito é de `glossario.md § Categorias de nó` —
> registrado aqui apenas como terceira fonte.

---

## 3. Menu de contexto (botão direito) e desativar nó

`[UI-OBSERVADA]` Botão direito sobre a seleção oferece: **Desativar**, **Duplicar selecionados**,
**Agrupar selecionados**. Funciona com um nó só ou com vários, e **tanto no modo mão quanto no
modo seta**. — `m3p1:56-61`

`[UI-OBSERVADA]` **Por nó**, os três pontinhos do próprio nó também têm **Desativar** / **Ativar**.
— `m3p1:352-353`, `m3p1:359-360`

`[UI-OBSERVADA]` **Como saber que um nó está desativado:** ele fica **sem cor**. — `m3p1:122`

`[VÍDEO]` Uso declarado de desativar: testar um caminho só. *"Eu preciso testar alguma coisa aqui,
e eu quero que ele só faça o caminho de clientes"* — `m3p1:120`.

`[UI-OBSERVADA]` **Duplicar** copia os nós selecionados **junto com as ligações entre eles**: a
aula duplica o par `Carrega Funcionarios → Insere funcionarios` e obtém a ramificação inteira
pronta para renomear. — `m3p1:54-66`

---

## 4. Grupos (agrupamento visual)

`[VÍDEO]` **Para que serve:** organizar fluxo grande sem criar subfluxo. *"Tem situações que você
não precisa criar um subfluxo, você só quer organizar melhor a tua tela"* — `m3p1:112`.
Demonstrado num fluxo real de produção (`PESSOAS - CORE` do ConstruShow), onde o grupo
`GERA NOVO IDPES` esconde uma lógica que *"se ele não tivesse dentro desse agrupamento […]
estaria aqui, tipo, tornando esse fluxo muito maior"* — `m3p1:108-110`.

### Criar

1. Selecione os nós (modo seta).
2. Clique em **agrupar** na barra esquerda — ou botão direito → **Agrupar selecionados**.
3. **Renomeie** o grupo (ex.: `Funcionarios`).
4. **(Opcional) Dê uma cor** ao grupo.
— `[UI-OBSERVADA]` `m3p1:76-95`

### O que o grupo passa a oferecer

`[UI-OBSERVADA]` `m3p1:96-152`:

| Recurso | Comportamento |
|---|---|
| **Colapsar / minimizar** | Recolhe o grupo numa caixa só |
| **Resumo no hover** | Grupo minimizado mostra **"N nós"** e **"N conexões"** |
| **Desagrupar** | Remove o grupo e joga os nós de volta para fora |
| **Desativar / Ativar** | **Cascata:** desativa *todos* os nós de dentro |
| **Focar no grupo** | Troca o canvas para dentro do grupo; você edita ali e clica em **Voltar** |
| **Redimensionar** | Arrastando a borda do grupo |

`[UI-OBSERVADA]` **"N conexões" conta as ligações que atravessam a fronteira do grupo, não as
internas.** Demonstrado: com 3 nós dentro, o resumo diz *"3 nós e 2 conexões"* — e o instrutor
explica: *"duas conexões pra fora do grupo, que é essa de entrada e essa de saída"*. — `m3p1:136-137`

`[UI-OBSERVADA]` **Nó dentro de grupo fica preso à área do grupo** — não sai arrastando.
— `m3p1:86-88`

`[UI-OBSERVADA]` **Armadilha prática:** expandir um grupo **exige espaço livre no canvas**; sem
espaço, ele se sobrepõe aos outros elementos e você tem que abrir espaço à mão.
*"Precisa de espaço aqui pra você expandir, né. Sempre ter essa ciência aqui."* — `m3p1:140-141`

`[VÍDEO]` **A tela salva sozinha enquanto você organiza:** *"e daí ele salva, vai salvando
conforme você vai organizando"* — `m3p1:142`.

> ⚠️ **`[LACUNA]` L33.** O autosave de `m3p1:142` foi dito só sobre reposicionamento. O instrutor
> clica em **Salvar** explicitamente depois de mudanças de configuração (`m3p1:202`, `m3p1:363`,
> `m3p1:495`). Falta descobrir **o que salva sozinho e o que exige o botão** — ver Lacunas.

---

## 5. Blocos de notas (`N`)

`[UI-OBSERVADA]` `m3p1:160-174`:

1. Clique em **Adicionar nota** (`N`) e posicione onde quiser no canvas — a nota é livre e
   fica fixa onde você soltar.
2. **Duplo clique** para editar: **título** + **descrição**.
3. Opções: **formatar texto**, **duplicar**, **cor** (demonstrado rosa), **deletar** (lixeirinha).

`[VÍDEO]` Propósito declarado: registrar **o porquê** de uma decisão de desenho. O exemplo escrito
na aula foi literalmente o título *"Por que fiz assim?"* com a descrição *"Foi separado
funcionários de clientes, pois no concorrente eram tabelas separadas"*. — `m3p1:162-166`

`[VÍDEO]` Quando a nota não basta, o caminho é a aba **Documentação** (seção 7): *"os blocos de
notas não serão suficientes, eu quero uma documentação"* — `m3p1:252`.

---

## 6. Configurar um nó e navegar entre nós

`[UI-OBSERVADA]` **Duplo clique no nó** abre o painel de parâmetros dele. — `m3p1:484-485`

`[UI-OBSERVADA]` Dentro do painel existem **atalhos de navegação entre nós** — botões
**Anterior** e **Próximo** — para pular de nó em nó *"sem precisar fechar e dar dois cliques,
fechar e dar dois clique"*. — `m3p1:486-488`

`[UI-OBSERVADA]` **Título e descrição do fluxo** são editáveis direto na tela: o título no canto
superior esquerdo, e a descrição clicando nela. — `m3p1:491-492`

---

## 7. As abas do topo

`[UI-OBSERVADA]` `m3p1:204-267`: **Editor**, **Copiloto**, **Execuções**, **Documentação**.

### Copiloto

`[UI-OBSERVADA]` `m3p1:204-227`:
- Abre um painel que pode ficar **fixado** à direita (padrão) ou **flutuante** — arrastável para
  qualquer lugar da tela.
- Flutuante pode ser **minimizado** numa janelinha de canto.
- **Histórico de conversas por fluxo** ("Ver conversas") e **Nova conversa**.
- **"Anexar barra lateral direita"** volta a fixar.
- `[VÍDEO]` Bug conhecido, dito em aula: o painel fixado **cobre os controles de zoom**. O
  instrutor declara que vai corrigir (`m3p1:223`).
- `[VÍDEO]` O que ele faz: *"ele vai te ajudando sobre a plataforma, ele vai criando fluxos pra
  você"*. Tem aula própria — fora do escopo deste lote. — `m3p1:206`, `m3p1:225`

### Execuções

`[UI-OBSERVADA]` `m3p1:228-249`. A lista traz, por execução: **horário**, **modo** (teste ou
produção) e **gatilho** (ex.: manual). Clicando numa execução, o painel central mostra:

- **status** (ex.: `COMPLETED`), **duração** (ex.: `2.8s`);
- **quem executou** e **onde** (ex.: espaço treinamento);
- **quantos nós tiveram sucesso** (ex.: `3/3`) e **quantas linhas processadas**.

`[UI-OBSERVADA]` **Ver resultados** por nó abre a tabela de saída daquele nó. **A prévia é de 100
linhas**, mesmo quando o nó leu 300 — o instrutor se corrige em voz alta sobre isso
(`m3p1:242-244`). Para o conteúdo completo há **download em CSV, Parquet e XLSX**
(`m3p1:245-246`).

`[UI-OBSERVADA]` **Não é todo nó que tem "Ver resultados".** O `Gravar Base Interna` não tem — e
o instrutor explica que não há necessidade, *"eu posso pegar no nó anterior"*: existem
*"alguns nós estratégicos"* que guardam resultado. — `m3p1:247-248`

`[UI-OBSERVADA]` O painel inferior mostra o **log em formato JSON** por nó, com o que executou e o
tempo. Tem botão **"Apenas conteúdo"** para esconder, e o painel **reabre sozinho na próxima
execução**. — `m3p1:477-482`

`[CONFIRMADO-DOC]` Dois complementos de `faq-perguntas-frequentes.md`:
- **Cancelar execução não exige fechar o navegador** — o painel inferior tem botão de cancelar;
  em fluxo agendado, vá em **Execuções** → execução em andamento → cancelar.
- **Os snapshots de execução são imutáveis:** *"mesmo se você editar o fluxo depois, o histórico
  mostra como estava quando rodou"*. Isso é **diferente** do histórico de versões — ver
  `versionar-e-exportar-fluxo.md`.

### Documentação

`[UI-OBSERVADA]` `m3p1:250-266`:
- Editor **markdown** com barra de formatação.
- Botão **"Gerar com IA"** — analisa o fluxo e escreve a documentação.
  `[VÍDEO]` **Falhou na aula** por chave de modelo não vinculada (*"eu tô usando Anthropic agora e
  nesse caso aqui eu não vinculei a chave"*) — `m3p1:254-256`. Ou seja: **depende de chave de
  provedor de IA configurada**.
- **Salvar** publica o texto, que passa a ser visível de fora pelo item **Ver documentação** do
  menu de contexto do fluxo na listagem.

`[CONFIRMADO-MCP]` A mesma documentação é legível por ferramenta: `ler_documentacao(workflow_id)`
devolve o markdown, o tamanho e se existe documentação.

---

## 8. Os botões do canto superior direito

`[UI-OBSERVADA]` `m3p1:269-333`.

| Controle | O que faz |
|---|---|
| **Executar** | Roda o fluxo |
| **Salvar** | Salva |
| **Seletor de execução** (mostra "Teste") | Abre menu com a seção **EXECUÇÃO** (Teste / Produção) e a seção **ALCANCE** |
| **Acesso e visibilidade** (ícone de dois usuários) | Modal de visibilidade e posse |

### EXECUÇÃO — Teste vs. Produção

`[VÍDEO]` **Teste roda uma janela de 500 linhas, sempre.** *"Por mais que a consulta que você
rode retorne 100.000 linhas, ele vai sempre executar 500. Porque teste é pra performance, é pra
você testar."* — `m3p1:275-276`

`[VÍDEO]` **Produção roda a totalidade.** — `m3p1:279-280`

> ⚠️ **Atenção de segurança (não é conflito, é consequência).** O modo Teste **limita volume, não
> suprime escrita**: na própria aula, uma execução em Teste **gravou 300 linhas** na tabela Oracle
> `FUNCIONARIOS` (`m3p1:438-443`). Teste **não é dry-run**. Isso confirma `lacunas.md` L2.

`[VÍDEO]` A segunda metade do par Teste/Produção é o **versionamento** — Teste roda o rascunho,
Produção roda a versão publicada. Está inteiro em `versionar-e-exportar-fluxo.md`.

### ALCANCE — toggle Template e "Somente usar"

`[UI-OBSERVADA]` `m3p1:281-326`:

- **Template** — marca o fluxo como molde. `[VÍDEO]` *"quando ele é template ele já se
  autocompartilha"* (`m3p1:285`). Efeito observado: o fluxo passa a aparecer **dentro dos
  projetos**, numa lista de **"fluxos públicos"**, com a etiqueta **TEMPLATE** e os botões
  **Clonar** e **Usar** (`m3p1:288-293`).
- **Somente usar (Uncloned)** — segundo toggle. `[VÍDEO]` Libera **Usar** e bloqueia **Clonar**:
  *"o Somente Usar ele só usa, ele não consegue ver as lógicas que têm internamente aqui […] e
  daí ele não consegue copiar"* (`m3p1:324-325`).
  Motivos declarados: fluxo desenvolvido pela Viasoft que o cliente não pode ver, cenário futuro
  de **venda de fluxos**, ou espaço de cliente cujo autor não quer expor a lógica aos próprios
  funcionários (`m3p1:322-323`).

`[VÍDEO]` **Clonar vs. Usar — a diferença dita em aula:** *"O clonar […] ele traz o fluxo pra cá,
ele copia, e daí você pode inserir, pode adicionar mais coisa, tratamento, personalizar. O Usar,
ele vai usar sempre de maneira intacta o que o original está replicando."* — `m3p1:317-318`.
Bate com `conceitos.md § Fluxo` `[CONFIRMADO-DOC]`, já registrado em `glossario.md`.

`[VÍDEO]` **O padrão real de uso do template na Viasoft** (vale como referência de arquitetura):
existe um fluxo `<ENTIDADE> - CORE` (ex.: `PESSOAS - CORE`) que *"sempre vai ser o mesmo"* — recebe
dados num contrato conhecido, trata e insere nas tabelas do ERP. Ele começa com um nó **Entrada do
Fluxo**. Cada concorrente ganha um fluxo de entrada próprio (ex.: `CARGA CLIENTES`, que consulta o
banco do concorrente `CISPIN`) e **chama o core como subfluxo** pelo nó `Chamar PESSOAS - CORE`.
*"A única coisa que vai mudar é a entrada."* — `m3p1:295-316`

### Acesso e visibilidade

`[UI-OBSERVADA]` `m3p1:328-332`. O modal permite:
- mudar a **visibilidade** — se está publicado, ou compartilhável **apenas com pessoas
  específicas** (exemplo dado: *"isso aqui só pode ser compartilhado com o Tony"*);
- **transferir a posse** para outra pessoa.

> ⚠️ **`[LACUNA]` L34 — o compartilhamento por pessoa contradiz o que se sabia.** `criar-fluxo.md`
> registra de `m1:60` que **"compartilhar com o time" significa todos do espaço, não um
> subconjunto**. Aqui, `m3p1:330-331` mostra escolha de **pessoa específica**. Provável leitura:
> são dois controles diferentes (visibilidade na criação × ACL fina depois), mas **não
> confirmado**. Ver Lacunas.

---

## 9. O menu ⋯ (três pontinhos)

`[UI-OBSERVADA]` Itens observados, na ordem em que aparecem na aula (`m3p1:335-476`):

| Item | O que é | Fonte |
|---|---|---|
| **Resumo do fluxo** | Painel com estatísticas: *"um resumo de tudo que tem dentro desse fluxo"* | `m3p1:337-339` |
| **Variáveis** | Painel de declaração de variáveis — ver `parametrizar-com-variaveis.md` | `m3p1:340-393` |
| **Capacidade MCP** | `[LACUNA]` Não demonstrado: *"isso aqui é utilizado mais avançado, não vamos ver isso agora"* | `m3p1:395-396` |
| **Tags** | Informar/corrigir a tag usada nos filtros da listagem | `m3p1:397-398` |
| **Sistema de Origem** | Qual sistema concorrente originou os dados | `m3p1:399-401` |
| **Histórico de versões** | Ícone de relógio com contador de alterações — ver `versionar-e-exportar-fluxo.md` | `m3p1:403-474` |
| **Publicar versão** | Publica; a plataforma calcula o número sozinha | `m3p1:415-418` |
| **Schema de E/S** | Entrada e saída; *"isso aqui é usado pra subfluxos"* | `m3p1:470-472` |
| **Importar / Exportar** | Aparece como "Importar" e "Exportar" **JSON/YAML** — ver `versionar-e-exportar-fluxo.md` | `m3p1:475-476` |

`[VÍDEO]` **"Sistema de Origem" está em vias de sair:** *"isso aqui provavelmente vai sair de
linha porque isso aqui era mais quando era focado em migração de dados"* — `m3p1:401`.

### Schema de E/S

`[VÍDEO]` *"Schema de IO […] que é input e output. Isso aqui é usado pra subfluxos."* — `m3p1:470-472`.
A explicação foi adiada na aula.

`[CONFIRMADO-DOC]` `guias-de-uso/nos/entrada-do-fluxo.md` fecha o sentido: o nó **Entrada do
Fluxo** (`workflow_input`) *"Expõe os parâmetros declarados no **input_schema**"*, e o trio do
padrão de subfluxo é **Entrada do Fluxo** + **Saída do Fluxo** (`workflow_output`) + **Chamar
Fluxo** (`call_workflow`). Ou seja: o **Schema de E/S é o contrato do subfluxo** — o que ele exige
receber e o que devolve.

`[CONFIRMADO-MCP]` `list_nodes(category='subflow')` devolve exatamente esses três nós, e a
descrição de `workflow_input` repete: *"Expõe os parâmetros declarados no input_schema para os nós
internos consumirem"*.

`[CONFIRMADO-DOC]` Limites que valem saber ao usar o Schema de E/S (`entrada-do-fluxo.md`):
- **Um só** nó Entrada do Fluxo por fluxo.
- O subfluxo roda em **contexto isolado** — não herda variáveis, conexões nem resultados do pai.
  **Exceção:** variáveis de **mesmo nome** no pai e no filho são propagadas automaticamente
  (auto-forward).
- Entrada do Fluxo **não dispara pelo botão Play** — só por `call_workflow`.
- Ciclo de chamada (A→B→A) é detectado e bloqueado (`call_stack` + `max_depth`).
- Rehidratação de dataset upstream mapeado carrega **até 1.000 linhas**; acima disso, use o nó
  **Loop (For Each)**.

---

## 10. Lacunas deste procedimento

`[LACUNA]` **L31 — atalhos não narrados.** Só `H`, `L`, `N`, `F`, `Ctrl+Z` e `Ctrl+Shift+Z` foram
ditos. Ficaram sem atalho conhecido: seta/seleção, agrupar, ajustar tela, organizar em grupos,
salvar, executar. **Impacto: baixo.**

`[LACUNA]` **L32 — "Capacidade MCP" no menu ⋯ não foi demonstrada.** É o único item do menu ⋯ sem
nenhuma cobertura. O nome sugere expor o fluxo como ferramenta MCP, o que seria relevante para
automação disparada por agente — mas isso é **suposição, não registro**. `m3p1:395-396`.
**Impacto: médio.**

`[LACUNA]` **L33 — o que o editor salva sozinho.** `m3p1:142` diz que reposicionar salva sozinho;
mas há cliques explícitos em **Salvar** após mudanças de configuração. Falta a fronteira exata —
importa porque salvar cria alteração pendente de publicação. **Impacto: baixo.**

`[LACUNA]` **L34 — dois modelos de compartilhamento de fluxo.** `m1:60` diz que compartilhar é
com **todo o espaço**; `m3p1:330-331` mostra compartilhar com **pessoa específica**. Falta saber
se são controles distintos ou se um substituiu o outro. **Impacto: baixo** para o piloto de
margem, **médio** para governança.

`[LACUNA]` **L35 — o nó "Grupo" não é nó.** `glossario.md` registra que `conceitos.md` cita "Nota"
e "Grupo" entre os nós, e que `list_nodes` não os devolve. Este lote confirma o palpite: nota e
grupo são **elementos de organização do canvas**, criados pela barra de ferramentas
(`m3p1:76-77`, `m3p1:160-161`), **não** arrastados da biblioteca. Fecha a parte "Nota"/"Grupo" de
L23. **Impacto: baixo.**
