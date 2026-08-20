# Procedimento: criar uma conexão a partir de um conjunto de arquivos

**Fonte:** `m2p3:181-212` `[UI-OBSERVADA]` / `[VÍDEO]`. Complemento com
`guias-de-uso/variaveis-e-arquivos-no-runtime.md` `[CONFIRMADO-DOC]` na parte de **arquivos que não
são conjunto**.

> ⚠️ **Aviso de cobertura.** **Nenhum documento oficial descreve "Conjunto" nem "A partir de um
> conjunto".** Varredura em `bruto/docs/**/*.md` por `conjunto de dados`, `a partir de um
> conjunto`, `file_set`, `fileset`: **zero ocorrências**. Todo o mecanismo abaixo vem de aula. Ver
> L-C1 no relatório do lote.

---

## 1. Para que serve

`[VÍDEO]` Dois cenários declarados — `m2p3:185-190`:

| Cenário | Formato | Citação |
|---|---|---|
| Sistema legado cujo "banco" é arquivo | **`.dbf`** | *"esses banco que eu já comentei […] que são arquivos esses .dbf por exemplo"* — `m2p3:185` |
| Cliente que manda planilha | **`.xlsx`**, **CSV** | *"muitas vezes o cara quer te mandar uma planilha"* — `m2p3:187` |

`[VÍDEO]` Prioridade declarada: *"planilha eu vou usar como exemplo, mas seria mais para situações
em bancos mais antigos como esses `.dbf`"* — `m2p3:190`.

`[VÍDEO]` **O ganho é poder usar SQL sobre arquivo:**

> *"você vai poder executar consultas em documentos como se fosse um banco de dados. Então você
> vai poder fazer SQL dentro de uma planilha. SQL dentro de um CSV. SQL dentro de um arquivo .dbf"*
> — `m2p3:211`

---

## 2. As duas formas de usar arquivo — não confundir

`[VÍDEO]` A distinção é feita explicitamente em `m2p3:204-205`:

| Pasta | Como se usa | Fonte |
|---|---|---|
| **Sem** a flag "Conjunto" | Os arquivos servem **dentro do fluxo**, pelos nós de entrada de arquivo | `[VÍDEO]` `m2p3:204`; `[CONFIRMADO-DOC]` modo **"Do projeto"** do nó CSV |
| **Com** a flag "Conjunto" | A pasta inteira pode virar **uma conexão**, consultável por SQL | `[UI-OBSERVADA]` `m2p3:194-206` |

> `[VÍDEO]` *"Se você cria só uma pasta desmarcada isso aqui, depois você vai poder usar lá dentro
> dos fluxos. Mas quando você cria uma pasta que é um conjunto de dados, aqui dentro das conexões
> você pode criar uma conexão a partir de um conjunto."* — `m2p3:204-205`

`[CONFIRMADO-DOC]` O caminho "sem conjunto" está documentado do lado do nó:
`variaveis-e-arquivos-no-runtime.md § Quando usar cada modo do Arquivo CSV` lista quatro modos do
campo Arquivo — **URL/Path**, **Do projeto** (*"arquivo já enviado antes"*), **Enviar**,
**Variável** — e explica que o upload devolve a referência **`shift-upload://<id>`**.

> ⚠️ `[CONFIRMADO-DOC]` **Variável tipo arquivo não funciona em fluxo agendado por cron** — *"não
> tem ninguém pra fazer upload"*. Para automação sem humano: modo **URL/Path** ou disparo por API
> com `variable_values` pré-resolvidos. — `variaveis-e-arquivos-no-runtime.md § Workflows
> agendados (cron)`

---

## 3. Passos

### Parte 1 — Criar a pasta marcada como Conjunto

`[UI-OBSERVADA]` `m2p3:191-197`:

1. **Entre no grupo econômico** (em aula: `Cliente XPTZ`) e abra o menu **Arquivos**.
   → `[UI-OBSERVADA]` O menu **Arquivos** também existe no nível do **espaço** — o instrutor abre
   Arquivos dentro do workspace `Treinamento` em `m2p3:184` antes de mudar para o cliente.
   **`[LACUNA]`** Falta confirmar se um conjunto criado no espaço gera conexão visível para todos
   os clientes abaixo (o modelo público/privado de `glossario.md § Conexão` sugere que sim, mas
   não foi demonstrado).
2. **Nova pasta.**
3. ☑ **Marque o checkbox "Conjunto".** É este o passo que habilita tudo o resto.
   > `[VÍDEO]` *"quando eu criar essa pasta, se esse cara for utilizado pra ser convertido em um
   > banco de dados, você tem que deixar marcado isso aqui"* — `m2p3:194`
4. **Nome.** Em aula: `SISHIFT`. `m2p3:196-197`
5. **Criar.**

> ⚠️ **`[LACUNA]` — o checkbox é reversível?** A aula não mostra editar uma pasta existente para
> marcar/desmarcar "Conjunto". Falta descobrir se dá para converter uma pasta comum em conjunto
> depois, ou se é decisão de criação (como o nível de dono da conexão). **Planeje como se fosse
> irreversível.**

### Parte 2 — Subir os arquivos

`[UI-OBSERVADA]` `m2p3:198-203`:

6. **Entre na pasta** e clique em **Incluir**.
7. Selecione os arquivos no explorador. Em aula: `PESSOA.xlsx` e `vendas.xlsx`. `m2p3:200-201`
8. O upload acontece e os arquivos aparecem listados dentro do conjunto. `m2p3:202-203`

### Parte 3 — Criar a conexão

`[UI-OBSERVADA]` `m2p3:205-208`:

9. Vá em **Conexões** (menu lateral, mesmo nível onde está o conjunto).
10. Clique em **"A partir de um conjunto"** — é uma **opção separada** de "+ Nova Conexão".
11. **Selecione o conjunto** (`SISHIFT`) e clique em **Criar conexão**.

---

## 4. O que essa conexão tem de diferente

`[UI-OBSERVADA]` `m2p3:209-211`:

| Recurso | Conexão de banco | Conexão de conjunto |
|---|---|---|
| **Testar conexão** | sim (ícone de raio) | **não** — *"ele não vai ter esse botão de testar a conexão porque ele é arquivo"* (`m2p3:209`) |
| **Playground** | sim | **sim** — *"ele vai ter o playground da mesma forma"* (`m2p3:210`) |
| Sistema obrigatório | sim (`m2p2:48`, `m2p3:141`) | **`[LACUNA]`** não mencionado |
| Ambiente / gate de produção | sim | **`[LACUNA]`** não mencionado |

`[VÍDEO]` O playground é a superfície de uso: SQL sobre planilha, CSV ou `.dbf`, *"como se fosse um
banco de dados"* — `m2p3:210-211`. O detalhamento do playground fica para aula posterior
(*"depois a gente vai verificar como que funciona o playground"*).

---

## 5. O que o MCP mostra (e não mostra)

`[CONFIRMADO-MCP]` `list_connections` neste ambiente devolve **só as duas conexões Oracle** — não
há nenhuma conexão de conjunto para inspecionar. A saída da ferramenta tem as colunas
`Nome / Tipo / Host / ID`, e o `Tipo` observado é sempre `oracle`.

`[CONFIRMADO-MCP]` O nó de extração **`sql_database`** exige `connection_id` descrito como *"UUID
do **conector SQL** de origem"*, e o "Use quando" cita apenas *"banco externo (PostgreSQL, SQL
Server, MySQL, Oracle)"*. **Conjunto de arquivos não aparece.**

`[CONFIRMADO-MCP]` **Não existe nó de leitura `.dbf`** no catálogo. `list_nodes` com busca `dbf`
retorna **nenhum resultado**. Os nós de arquivo disponíveis na categoria `input` são:
`csv_input`, `excel_input` (*"o formato antigo .xls (97-2003) não é suportado"*), `ofx_input`,
`xml_input`, `inline_data`, `internal_data_source`, `google_drive_download`,
`google_sheets_input`, `http_request` e `sql_database`.

> ⚠️ **`[LACUNA]` de alto valor — como se LÊ um conjunto dentro de um fluxo?**
> A aula prova que o conjunto vira conexão e que a conexão tem playground. **Não mostra nenhum nó
> consumindo essa conexão.** As três hipóteses, nenhuma confirmada:
> 1. `sql_database` aceita o `connection_id` do conjunto (a descrição diz "conector SQL", o que
>    depõe contra);
> 2. existe um nó específico não identificado;
> 3. só o playground consome, e o fluxo usa `csv_input`/`excel_input` apontando para os arquivos.
>
> **Isso importa porque é o único caminho conhecido para ler `.dbf` na plataforma** — não há nó de
> `.dbf`. Falta descobrir criando um conjunto de teste e olhando o campo de conexão dos nós (o que
> exigiria escrita, **fora do permitido neste lote**).

`[CONFIRMADO-MCP]` Detalhe correlato: `google_drive_download` *"devolve a referência
`shift-upload://<id>` que os nós de entrada de arquivo (CSV, Excel) leem"* — confirmando que
`shift-upload://` é o esquema de referência a arquivo da plataforma, o mesmo do modo "Do projeto".

---

## 6. Limites de upload — e a divergência do `.dbf`

`[CONFIRMADO-DOC]` `faq-perguntas-frequentes.md § Limites de upload`:

| Limite | Default | Configurável? |
|---|---|---|
| Tamanho por arquivo | **500 MB** | Sim, via env |
| Quota total por projeto | **5 GB** | Sim, via env |
| TTL (arquivos não usados) | **30 dias** | Sim, via env |
| Extensões aceitas | **`.csv`, `.tsv`, `.xlsx`, `.xls`, `.json`, `.parquet`, `.txt`** | **Não** |

`[CONFIRMADO-DOC]` *"Arquivos não acessados há mais de 30 dias são removidos automaticamente. Se um
fluxo agendado usar o arquivo, isso atualiza o 'último acesso' e protege da limpeza."*

> ⚠️ **DIVERGÊNCIA D-C1 — `.dbf` não está na lista de extensões aceitas.**
> - `[VÍDEO]` Aula: o caso de uso **principal** do conjunto é `.dbf`. O instrutor abre uma pasta
>   cheia de `.dbf` (`m2p3:185-186`) e promete *"SQL dentro de um arquivo `.dbf`, que é no caso
>   daquele cliente ali que tem aqueles arquivos `.dbf`"* (`m2p3:211`).
> - `[CONFIRMADO-DOC]` FAQ: a lista de extensões aceitas **não inclui `.dbf`** e é marcada como
>   **não configurável**.
>
> **Registrado sem escolher.** Hipóteses, nenhuma verificada: (a) a lista do FAQ cobre o upload de
> arquivo para nó e o conjunto tem outra lista; (b) `.dbf` foi adicionado depois do FAQ; (c) o
> `.dbf` da aula era aspiração, não recurso pronto. **Impacto:** se `.dbf` não sobe, o caso de uso
> declarado como principal do conjunto **não funciona**. Confirmar antes de prometer a cliente.

> ⚠️ **DIVERGÊNCIA D-C2 — `.xls` sobe mas nenhum nó lê.** `[CONFIRMADO-DOC]` O FAQ aceita `.xls`
> no upload; `[CONFIRMADO-MCP]` `excel_input` declara que *"o formato antigo .xls (97-2003) **não é
> suportado**"*. Dá para subir um arquivo que nenhum nó de Excel consome.

---

## 7. Lacunas específicas deste procedimento

1. **Quais extensões um CONJUNTO aceita.** A lista do FAQ é do upload de arquivo; não há lista
   específica do conjunto. Ver D-C1.
2. **Como cada arquivo se torna tabela.** O nome da tabela vem do nome do arquivo? Da aba?
   → `[CONFIRMADO-DOC]` No mundo dos **nós**, multi-aba é resolvido com **um nó por aba**:
   *"você cria 2 nós Excel apontando pro mesmo arquivo, cada um com aba diferente"*
   (`modelos-de-entrada.md § Excel multi-sheet`). **`[LACUNA]`** Nada diz como o **conjunto**
   trata uma planilha com várias abas — se vira N tabelas ou só a primeira.
3. **Tipagem das colunas.** Planilha não tem esquema; `excel_input` materializa em DuckDB
   `[CONFIRMADO-MCP]`, mas não se sabe se o conjunto usa o mesmo motor nem como infere tipos.
4. **Número de arquivos por conjunto.** O FAQ dá tamanho e quota, não a contagem.
5. **Atualização.** Substituir um arquivo dentro do conjunto invalida a conexão? Recria as tabelas?
   Precisa recriar a conexão?
6. **Modelo de entrada.** `[CONFIRMADO-DOC]` Modelos de Entrada validam o cabeçalho **no nó**
   CSV/Excel (`modelos-de-entrada.md § Como vincular`, campo `input_model_id`). **`[LACUNA]`** Não
   se sabe se um conjunto pode ser validado por Modelo de Entrada.

---

## 8. Observações para o piloto

1. **Não é o caminho do piloto de margem.** Os pedidos vivem em Oracle e há duas conexões diretas
   funcionando `[CONFIRMADO-MCP]`. O conjunto de arquivos é caminho de **dado que chega de fora**.
2. **Onde ele pode entrar de verdade:** uma **tabela de parâmetros de preço/margem mantida em
   planilha** pelo time comercial. Nesse caso, atenção ao aviso `[CONFIRMADO-DOC]`: **variável tipo
   arquivo não funciona sob cron.** Para um fluxo agendado, o arquivo tem de estar em **URL/Path**
   ou já enviado ("Do projeto"/conjunto) — não pode depender de upload na hora da execução.
3. **Se a planilha de parâmetros for recorrente**, a alternativa mais firme é uma **Base de Dados
   interna** (`internal_data_source` / `internal_data_write`, `[CONFIRMADO-MCP]`, sem conexão
   externa) — é o tema da parte 4 do módulo 2 (`m2p3:213`), fora deste lote.
4. **Cuidado com o TTL de 30 dias** `[CONFIRMADO-DOC]`: arquivo não acessado é apagado
   automaticamente. Um conjunto usado por fluxo **agendado** está protegido (o uso atualiza o
   "último acesso"), mas um conjunto de parâmetros usado só manualmente pode desaparecer.
   **`[LACUNA]`** Não se sabe se o TTL se aplica igual dentro de uma pasta marcada como Conjunto.
5. **A conexão de conjunto não tem "Testar conexão"** `[UI-OBSERVADA]`, e o MCP não oferece
   equivalente: `diagnosticar_conexao` diagnostica por etapas de rede (`dns`/`tcp`/`auth_query`),
   que não existem para arquivo. **`[LACUNA]`** Sem forma conhecida de validar uma conexão de
   conjunto por ferramenta — só pelo playground, na mão.
