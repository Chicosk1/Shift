# Procedimento: criar uma conexão Oracle via TNS

**Fonte:** `m2p3:1-46` `[UI-OBSERVADA]` / `[VÍDEO]`. Complemento de
`procedimentos/configurar-conexao-direta.md` passo 8b.

> ⚠️ **Aviso de cobertura.** **Nenhum documento oficial menciona TNS.** Varredura em
> `bruto/docs/**/*.md`: a única ocorrência da sigla é `extracao-na-borda-design.md:16`, e ali
> significa *o protocolo* Oracle, não o cadastro de conexão. Ou seja, **todo este procedimento
> vem de aula** — não há `[CONFIRMADO-DOC]` em nenhuma linha abaixo. Ver L-T1 no relatório do
> lote.

---

## Quando usar TNS em vez de Easy Connect

`[VÍDEO]` TNS é **o caminho da cloud Viasoft**. O motivo declarado em aula:

> *"isso aqui tá na nossa cloud, mas ele não tá na nossa rede interna, ele é uma rede separada,
> lá dentro da Oracle, dedicada aos clientes"* — `m2p3:39`

`[VÍDEO]` E por isso IP + porta **não** resolve: *"esse banco ele é o IP mas tem toda uma questão
de redirecionamento interno que acontece nessas bases que são de clientes que são cloud"* —
`m2p3:42`. O instrutor chega a rodar `ipconfig` para mostrar que pegar o IP local não ajudaria
(`m2p3:40-41`).

`[VÍDEO]` **Regra de decisão declarada — o SQL Developer é o norte:** `m2p3:43-46`

| O que o SQL Developer mostra na conexão | O que criar no Shift |
|---|---|
| Tipo **TNS** (com *network alias*) | Conexão por **TNS Description** |
| Tipo **Basic** | Conexão por **Easy Connect** (host/porta/banco) |

> *"Se aqui tiver definido como TNS, você tem que criar por TNS lá dentro do Shift. […] Se tiver
> aqui Basic, daí você pode criar Basic."* — `m2p3:45-46`. Com a ressalva do próprio instrutor:
> *"se for nosso cloud, obviamente"*.

`[VÍDEO]` A conexão TNS resultante **é considerada direta** — não passa por relay: *"eu consegui
fazer uma conexão via TNS de maneira direta também"* — `m2p3:38`.

---

## Parte 1 — Extrair o descriptor do `tnsnames.ora`

`[VÍDEO]` O problema de partida: a tela de propriedades do SQL Developer mostra **só o alias de
rede**, não o descriptor. Em aula o alias era `DECO`.

> *"Ela tá aqui: TNS, e o network alias DECO. Só que aqui só tem isso. Como que eu vou saber qual
> que é a description?"* — `m2p3:12-13`

### Passos

1. **Localize o `tnsnames.ora` do Oracle Client.** `[VÍDEO]` `m2p3:18-22` — caminho descrito em
   aula, na ordem em que foi navegado:

   ```
   C:\app\  →  client…  →  client32  →  viasoft  →  product  →  <versão do Oracle>
             →  client_1  →  network  →  admin  →  tnsnames.ora
   ```

   `[VÍDEO]` *"Geralmente as instalações do Oracle, no padrão que a gente usa, elas vão ficar aqui
   em APP"* — `m2p3:18`. E o que está ali **não é o banco**, são *"as configurações de conexão via
   TNS"* — `m2p3:20`.

   `[VÍDEO]` **Truque para descobrir se é client32 ou client64:** a pasta que existe é a que está
   em uso. *"Ela vai ficar aqui geralmente client32, se você não achar ou não tiver o… se eu
   tentar ir pra frente aqui eu nem consigo acessá-lo. Então ele tá usando o 32."* — `m2p3:21`

   > **`[LACUNA]`** O prefixo exato do caminho não é legível na transcrição — a marcação de cena
   > registra `C:\app\client…` (`m2p3:19`) e a fala emenda `client32, viasoft, product`
   > (`m2p3:22`). Falta confirmar se é `C:\app\client\…\client32\…` ou `C:\app\client32\…`, e
   > qual é o segmento entre `client` e `client32`. Não inventar: **conferir na máquina**.

2. **Abra o arquivo em editor de texto.** `[VÍDEO]` Em aula, Notepad++ pelo menu de contexto
   ("Edit com Notepad++"). `m2p3:24`

3. **Copie APENAS o bloco `DESCRIPTION`.** Esta é a instrução mais explícita do trecho:

   > `[VÍDEO]` *"Então aqui tá o DECO lá, e a description. Então eu sempre vou pegar a partir da
   > description, **nunca o nome junto** o DECO."* — `m2p3:23`

   Ou seja: no arquivo o conteúdo tem a forma `ALIAS = (DESCRIPTION = …)`. **O alias fica fora**;
   só o `(DESCRIPTION = …)` vai para o Shift.

   > **`[LACUNA]`** O texto literal do descriptor não aparece na transcrição (foi copiado na tela,
   > não lido em voz alta). Falta um exemplo real de descriptor da cloud Viasoft — com `HOST`,
   > `PORT`, `SERVICE_NAME` e eventual `LOAD_BALANCE`/`FAILOVER` — para servir de gabarito.

---

## Parte 2 — Criar a conexão no Shift

`[UI-OBSERVADA]` `m2p3:25-37`. Os campos são os mesmos da conexão direta
(`configurar-conexao-direta.md`), **mudando só o passo 8**.

| # | Campo | O que foi preenchido em aula | Linha |
|---|---|---|---|
| 1 | Nome | `ConstruShow TNS` | `m2p3:25` |
| 2 | Sistema | `ConstruShow` | `m2p3:27` |
| 3 | Tipo de banco | `Oracle` (vem do sistema) | `m2p3:27` |
| 4 | **Ambiente** | `Homologação` | `m2p3:28-29` |
| 5 | Usuário | `VIASOFTMCP` | `m2p3:28-30` |
| 6 | Senha | `VIASOFTMCP` | `m2p3:31` |
| 7 | **Formato de conexão** | trocado para **`TNS Description`** | `m2p3:32` |
| 8 | **`TNS Descriptor`** | colado o bloco extraído do `tnsnames.ora` | `m2p3:32-33` |
| 9 | **Schemas adicionais** | `VIASOFTMCP`, `VIASOFTBASE`, `VIASOFTFIN` | `m2p3:33-34` |
| 10 | — | **Criar** → *"Criou com sucesso"* | `m2p3:35-36` |
| 11 | — | **Testar conexão** → sucesso | `m2p3:36-37` |

`[UI-OBSERVADA]` Rótulos literais lidos na tela: **`TNS Description`** (opção do seletor de
formato de conexão), **`TNS Descriptor`** (o campo de texto onde se cola), **`Schemas
adicionais`**.

> ⚠️ **Discrepância de rótulo entre lotes — registrada, não resolvida.**
> `m2p2:27-33` (lote 2) chama o campo de **"Esquemas"**; `m2p3:33` chama de **"Schemas
> adicionais"**. Pode ser a mesma caixa descrita de dois modos, pode ser um campo diferente
> ("adicionais" sugere *além* dos do usuário). **`[LACUNA]`** Falta confirmar na tela se são um
> ou dois campos.

`[UI-OBSERVADA]` A conexão TNS **tem** o botão de testar conexão, como a direta. `m2p3:36-37`

`[UI-OBSERVADA]` Conexão TNS é excluível pelo ícone de lixeira com confirmação — o instrutor apaga
uma `ConstruShow TNS` preexistente no início da aula. `m2p3:6`

---

## O que o MCP mostra (e o que não mostra)

`[CONFIRMADO-MCP]` As duas conexões Oracle deste ambiente respondem `status: ok` e
**`via_relay: false`** em `diagnosticar_conexao`, com as três etapas (`dns`, `tcp`, `auth_query`)
verdes:

| Conexão | Host | Porta | Banco | `via_relay` | Latência `auth_query` |
|---|---|---|---|---|---|
| Viasuper Padrão - Gabriel | 192.168.90.218 | 30200 | ORCL | `false` | 363 ms |
| Viasuper Titan - Gabriel | 192.168.90.218 | 30100 | ORCL | `false` | 269 ms |

`[INFERIDO]` As duas são **Easy Connect**, não TNS: `get_connection` devolve `Host`, `Porta` e
`Banco` preenchidos e coerentes entre si, o que é a forma do Easy Connect.

> **`[LACUNA]`** **`get_connection` não devolve o método de conexão.** Não há campo
> `connection_method`, `tns_descriptor` nem equivalente na saída. Portanto não é possível
> **confirmar** pelo MCP se uma conexão é TNS ou Easy Connect, nem inspecionar o descriptor de
> uma conexão TNS existente. Falta descobrir como uma conexão TNS aparece em `get_connection` —
> se `Host`/`Porta`/`Banco` vêm vazios, derivados do descriptor, ou algo mais.

---

## Observações para o piloto

1. **O piloto provavelmente não precisa de TNS.** `[CONFIRMADO-MCP]` As duas conexões Oracle do
   ambiente já conectam direto, sem relay e sem TNS (`via_relay: false`, `auth_query` ok). TNS
   entra em cena **se e quando** o banco de pedidos migrar para a cloud Viasoft — o critério é o
   do `m2p3:45`: olhar o tipo da conexão no SQL Developer.
2. **Se precisar, o custo é baixo, mas há uma dependência humana:** o descriptor precisa ser
   extraído de uma máquina que **tenha Oracle Client instalado** com o `tnsnames.ora` já
   configurado para aquele banco. Não é algo que se descubra pelo Shift nem pelo MCP.
3. **`Ambiente` continua sendo o campo que decide quem executa** — em aula a conexão TNS foi
   criada como `Homologação` (`m2p3:28`), fora do gate de produção. Ver
   `configurar-conexao-direta.md § Observações para o piloto`.
4. **Schemas:** a aula repete a mesma tríade `VIASOFTMCP` / `VIASOFTBASE` / `VIASOFTFIN` da
   conexão direta. As conexões reais do piloto usam o usuário `VIASOFTMERC` — **`[LACUNA]`** não
   se sabe quais schemas foram habilitados nelas, e `get_connection` não devolve essa lista.
