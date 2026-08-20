# Procedimento: primeiro fluxo — CSV para tabela de banco

O fluxo mais comum do Shift, ponta a ponta. **Duas fontes independentes descrevem o mesmo
procedimento**, e elas concordam: `primeiros-passos.md` `[CONFIRMADO-DOC]` (destino Postgres) e
`m1:64-76` `[VÍDEO]` / `[UI-OBSERVADA]` (destino Oracle, fluxo `Importar Funcionários`).

**Desenho:** `Manual → CSV → Mapeamento → Inserção em Massa`
(`primeiros-passos.md §5` omite o gatilho e descreve `CSV → (Mapeamento) → Inserção em Massa`.)

---

## Pré-requisitos

`[CONFIRMADO-DOC]` — `primeiros-passos.md § O que você vai precisar`

1. Estar dentro de um **projeto**.
2. Uma **conexão** com o banco de destino já cadastrada.
3. O **arquivo CSV**.
4. Uma **tabela de destino já existente** no banco, com colunas compatíveis.

> `[VÍDEO]` Em aula a tabela foi criada na hora, à mão, com `CREATE TABLE` no gerenciador de
> banco, e conferida com `SELECT *` (vazia) antes de seguir. — `m1:71`
> **O Shift não cria a tabela de destino.**

---

## Passos

### 1. Crie o fluxo
Ver `procedimentos/criar-fluxo.md`. → Você cai no **canvas**.

### 2. Adicione o gatilho
`[UI-OBSERVADA]` Abra a **biblioteca de nós** — atalho **`L`** alterna abrir/fechar. Adicione o
nó de gatilho **Manual**.
→ A biblioteca de gatilhos mostra: Manual, Agendamento, Webhook, Monitoramento, Entrada.
Nenhuma configuração é necessária. — `m1:64-65`

### 3. Adicione e configure o nó CSV
`[CONFIRMADO-DOC]` Adicione o nó **CSV** (grupo *Entradas*) e configure:

- **Arquivo** — upload, ou aponte URL/caminho.
- **Delimitador** — `,` por padrão. Troque se o arquivo usa `;`.
- **Cabeçalho** — deixe ligado se a primeira linha tem os nomes das colunas.

`[UI-OBSERVADA]` As cinco origens possíveis: link, arquivos do projeto, arquivos da área, enviar
agora, ou **variável** (para arquivo que chega por API). — `m1:66`

`[VÍDEO]` Em aula, delimitador e encoding foram mantidos no padrão porque o arquivo já era
vírgula e UTF-8. — `m1:66`

### 4. Rode o preview e confira
`[CONFIRMADO-DOC]` Rode o **preview** do nó e confira que as linhas aparecem como esperado.
— `primeiros-passos.md §2`

`[VÍDEO]` Duas formas de executar: pelo botão Executar do canvas, ou dando duplo clique no nó e
executando **dentro da configuração** — neste caso ele **roda os nós anteriores até aquele**.
→ O painel inferior mostra os **logs**: cada passo, o que aconteceu e o tempo. O resultado é
visualizável **em tabela**. — `m1:68-69`

> `[CONFIRMADO-DOC]` Se quem fornece o arquivo costuma errar nome de coluna, vincule um
> **Modelo de Entrada**: valida **antes** de processar e falha com mensagem clara em vez de erro
> críptico adiante. — `primeiros-passos.md §2`

### 5. (Opcional) Alinhe as colunas
`[CONFIRMADO-DOC]` Se os nomes do CSV não batem com os da tabela, adicione **Mapeamento** (grupo
*Transformação*) entre o CSV e a saída. Pode pular se já coincidem. — `primeiros-passos.md §3`

`[VÍDEO]` Em aula o objetivo foi outro — **transformar `nome` em maiúsculo**, por ser padrão do
sistema de destino. O nó foi renomeado para "Transforma em Maiúsculos". Duas estratégias
mostradas:
- **"mapear todos"** — mapeamento automático de todas as colunas; ou
- mapear **só a coluna a transformar** e marcar a opção de passar adiante as não incluídas.

→ O nó **carrega o schema do nó anterior** automaticamente. Após executar, a coluna transformada
aparece **no fim** do dataset. — `m1:70`

### 6. Adicione e configure a Inserção em Massa
`[CONFIRMADO-DOC]` Adicione **Inserção em Massa** (grupo *Banco de Dados*) e configure:

- **Conexão** — a conexão do pré-requisito.
- **Tabela de destino**.
- **Mapeamento de colunas** — quais colunas da entrada vão para quais da tabela.
- **Estratégia de carga** — comece com `append_fast` (padrão, só insere).

`[UI-OBSERVADA]` Existe **auto-mapear**, que reaproveita o schema já conhecido de uma execução
anterior. — `m1:73`

> `[VÍDEO]` O instrutor usou **Insert**, mencionando que **upsert** serviria *"pra ele verificar
> se já existe"*, mas escolheu insert por rapidez. — `m1:73`
> Para carga que roda repetidas vezes, `upsert` ou `insert_if_not_exists` evitam duplicar — e
> **exigem indicar a coluna que identifica um registro único** `[CONFIRMADO-DOC]`.

> ⚠️ `[CONFIRMADO-MCP]` **Se o destino for Firebird, pare aqui:** o nó não escreve em Firebird,
> e a falha só aparece na primeira execução. Ver `nos/bulk-insert.md`.

**Alternativa sem banco externo** `[CONFIRMADO-DOC]`: troque pelo nó **Base de Dados Interna
(Escrita)**, criando antes a base em *Bases de Dados* (menu do espaço). Modo inicial *insert*.
Bom para testar sem conexão, mas **teto de 200 mil linhas**. — `primeiros-passos.md §4`

### 7. Conecte os nós
`[CONFIRMADO-DOC]` Arraste da **bolinha de saída** de um nó até a **bolinha de entrada** do
próximo, nesta ordem: **CSV → (Mapeamento) → Inserção em Massa**. — `primeiros-passos.md §5`

`[VÍDEO]` Lembrete de forma: **só o gatilho não tem entrada** — todos os outros têm entrada e
saída. — `m1:73`

### 8. Salve e execute
`[UI-OBSERVADA]` **Salvar** e **Executar** são os dois botões principais do topo do canvas.
— `m1:63`

→ `[CONFIRMADO-DOC]` Ao terminar, o painel mostra o **`row_count`** — quantas linhas foram lidas
e gravadas. Se algo falhar, a mensagem aponta **o nó e o motivo** (coluna ausente, tipo
incompatível). — `primeiros-passos.md §6`

`[VÍDEO]` Resultado da aula: **300 funcionários em ~2 segundos**. — `m1:74`

### 9. Confirme no destino
`[VÍDEO]` Rode um `SELECT *` na tabela pelo gerenciador de banco.
→ As 300 linhas devem estar lá. — `m1:75`

---

## Observações

`[CONFIRMADO-DOC]` Próximos passos sugeridos pela própria doc: o nó CSV tem mais opções
(encoding, retentativa, limite de linhas); parametrizar o fluxo com **Variáveis e arquivos** para
o arquivo mudar a cada execução; garantir qualidade com **Modelos de Entrada**.
— `primeiros-passos.md § E agora?`

`[LACUNA]` O procedimento cobre **desenvolver e executar manualmente**. Não cobre **publicar**,
**agendar** nem **monitorar** — que é o que o piloto precisa. Previsto nos lotes 4 e 5.
