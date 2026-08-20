# Procedimento: configurar um relay (conexão de borda / banco on-premise)

**Fontes:**
- Aula: `m2p3:47-179` `[UI-OBSERVADA]` / `[VÍDEO]` — a demonstração ponta a ponta no Windows.
- Doc: `instalar-o-relay-no-windows.md` e `instalar-o-relay-no-linux.md` `[CONFIRMADO-DOC]` —
  manuais de instalação de estado corrente (não são documentos de proposta; não estão na lista de
  design de `lacunas.md` L17).
- MCP: `diagnosticar_conexao`, `describe_node sql_database` `[CONFIRMADO-MCP]`.

> ⚠️ **Este procedimento tem 7 divergências aula↔doc**, todas marcadas em linha com **D-Rn** e
> repetidas no fim. Duas delas mudam o que você digita (**D-R1** e **D-R2**) e uma muda se a
> instalação funciona (**D-R3**). Leia-as antes de executar.

---

## 1. O que o relay é, e quando entra

`[CONFIRMADO-DOC]` `instalar-o-relay-no-windows.md § 1` — o `shift-relay` é o componente que roda
**na infra do cliente** e permite que o Shift (nuvem) alcance bancos on-premise **sem abrir
nenhuma porta de entrada no firewall**.

```
  Rede do cliente                              Nuvem
  ┌──────────────┐     ┌─────────────┐        ┌───────────────┐
  │ banco(s)     │◄────│ shift-relay │ ──wss─►│ shift-gateway │◄── Shift
  │ ERP/Oracle/… │ LAN │  (serviço)  │ saída  │               │
  └──────────────┘     └─────────────┘  443   └───────────────┘
```

Quatro propriedades declaradas `[CONFIRMADO-DOC]` (`§ 1` dos dois manuais):

| Propriedade | O que significa na prática |
|---|---|
| **Só tráfego de saída** | O relay sempre inicia a conexão; o gateway nunca alcança a rede do cliente por conta própria |
| **Burro de propósito** | Não conhece tipo de banco, não tem driver, **não vê usuário/senha/SQL do fluxo** — para ele é tudo TCP. A definição da conexão vive no Shift |
| **Allowlist local** | Só alcança os `host:porta` declarados no config dele. **Não é proxy genérico** |
| **Um binário, sem dependências** | Go estático: sem JVM, sem .NET, **sem Oracle Instant Client** |

`[CONFIRMADO-DOC]` **Um relay por rede isolada.** Filiais/VLANs sem rota entre si exigem um relay
cada. Bancos na mesma LAN são atendidos por um relay só — basta listar os dois destinos.
— `instalar-o-relay-no-windows.md § 1`

`[VÍDEO]` A regra prática de aula: *"Nós vamos ter que sempre instalar na máquina lá no ambiente
do cliente em uma máquina que tenha acesso ao banco de dados, senão não adianta"* — `m2p3:61`.
Vale para Linux ou Windows (`m2p3:62`).

`[VÍDEO]` Em aula a demonstração foi **simulada**: a máquina do instrutor, dentro da rede Viasoft,
fez o papel de on-premise. *"Por mais que ela tá na minha rede, vocês vão ver que vai funcionar
igual"* — `m2p3:50-51`. Isso importa na leitura: **nada no trecho prova o comportamento atravessando
um firewall de cliente real.**

---

## 2. Antes de começar

### 2.1 Onde criar o relay: no grupo econômico

`[VÍDEO]` `m2p3:69-70` — a orientação é categórica:

> *"Isso vale pra espaço também funciona. Mas geralmente você vai, quando é comunicação de borda,
> sempre vai ser dentro do cliente, porque se você tá fazendo uma comunicação de borda é um banco
> de dados quente."*

`[VÍDEO]` E o motivo do risco, repetido no fecho (`m2p3:170-173`):

> *"Sempre configurar a nível de cliente. […] porque se você conectar a nível de espaço, todos vão
> poder acessar essa conexão. […] pra você não ter nenhum acidente de colocar uma conexão quente
> de cliente no espaço e daí alguém tá testando usa aquela conexão."*

`[UI-OBSERVADA]` Demonstrado por contraste: criado um segundo grupo econômico (`Space X`), suas
Conexões aparecem **vazias** — a conexão do `Viasoft Soluções` não vaza para ele. `m2p3:174-178`

> **D-R7 — divergência de navegação.** A aula acessa **Relays como item de menu do grupo
> econômico**, sob "parte de conectividade" (`m2p3:71-73`), e também encontra a tela em
> **Configurações → Relays** no nível do espaço (`m2p3:90-95`). Os dois manuais só citam
> **"Configurações → Relays"** (`instalar-o-relay-no-windows.md § 3`;
> `instalar-o-relay-no-linux.md § 2 Passo 0`). Provável que existam as duas entradas para o mesmo
> recurso em níveis diferentes, mas **`[LACUNA]`**: falta confirmar se o relay criado em
> Configurações (espaço) e o criado no grupo econômico são o mesmo objeto com escopos diferentes.

`[UI-OBSERVADA]` O ícone do menu Relay é **uma tomadinha** — o mesmo símbolo de Conexões:
*"ele é uma tomadinha que vai ligar o Shift com o banco de dados"*. `m2p3:73`

### 2.2 Papel necessário

`[CONFIRMADO-DOC]` **ADMIN** do workspace/grupo econômico para criar relay e para usar **Baixar
instalador**. — `instalar-o-relay-no-windows.md § 3 e § 4`; `instalar-o-relay-no-linux.md § 2`
`[VÍDEO]` A aula não menciona papel. Não é conflito — é silêncio.

### 2.3 Requisitos da máquina do cliente — Windows

`[CONFIRMADO-DOC]` `instalar-o-relay-no-windows.md § 2`:

| Item | Requisito |
|---|---|
| Sistema | Windows x64 (Windows 10/11 ou Windows Server 2016+) |
| Permissão | **Administrador local** (o instalador registra um serviço) |
| Disco | ~100 MB de programa + espaço na **pasta de dados** (logs e os Parquets da borda, que podem ser grandes) |
| Rede | **Saída HTTPS/443** liberada para `tunnel-shift.viasoftcloud.com.br` |
| Relógio | Sincronizado (NTP). Skew grande atrapalha TLS e é sinalizado pelo `diag` |

> ⚠️ `[CONFIRMADO-DOC]` **Proxy corporativo explícito obrigatório não é suportado.** O relay
> **não** usa `HTTP_PROXY`/`HTTPS_PROXY` para a conexão `wss` — é preciso liberar saída direta na
> 443 para o host do gateway. — `§ 2`

### 2.4 Requisitos — Linux

`[CONFIRMADO-DOC]` `instalar-o-relay-no-linux.md § 2`: qualquer distro com **systemd**
(RHEL/Oracle Linux 8+, Debian 11+, Ubuntu 20.04+, SUSE), `x86_64` (há build `arm64`), `root`/sudo,
~100 MB + `data_dir`, saída 443, relógio sincronizado.
`[CONFIRMADO-DOC]` **Não é preciso liberar nada no `firewalld`** — o relay não escuta porta.

> **D-R6 — divergência sobre existir Linux.** `[VÍDEO]` `m2p3:99`: *"Aqui tem o instalador, por
> enquanto ele tem o Windows. Mas já vai ser disponibilizado nas próximas versões pra instalação
> em servidores Linux também."* Mas `instalar-o-relay-no-linux.md` **existe e é um manual
> completo**, e `instalar-o-relay-no-windows.md:9` linka para ele. `[INFERIDO]` A aula é anterior
> ao manual. **Não é o mesmo produto de embalagem:** o manual Linux diz explicitamente que **não
> há `Setup.exe`, wizard nem bundle `.zip`** — no Linux *"o instalador é o próprio binário:
> `shift-relay install` escreve a unit do systemd"* (`§ intro`). Ou seja, o que a aula diz que
> "vai ser disponibilizado" nunca foi um wizard.

---

## 3. Passo 1 — Criar o relay no Shift e copiar a chave

`[UI-OBSERVADA]` `m2p3:74-80`:

1. **Relays → `+ Novo Relay`.**
2. **Nome.** Em aula: `Local`.
   → `[VÍDEO]` Alerta dado na hora: *"tem que ser um nome que depois você vai usar"* — `m2p3:74`.
   Ver **D-R3** abaixo: esse nome reaparece no instalador.
3. **Criar.**
4. **Copie a chave.** `[UI-OBSERVADA]` *"Ele vai gerar essa chave aqui, é uma chave única e você
   só consegue pegar ela uma vez, então aqui até diz: uma única vez."* — `m2p3:78`. A própria tela
   avisa. Em aula a chave foi colada no Notepad++ para não se perder. `m2p3:79-80`
5. **Concluir.**

`[CONFIRMADO-DOC]` Confirma e complementa: o **token** é *"exibido uma única vez; se perder, crie
outro relay (ou revogue e recrie)"*. — `instalar-o-relay-no-windows.md § 3`

`[UI-OBSERVADA]` A chave pode ser **revogada ou excluída** depois: *"essa chave aqui você pode ou
revogar ela ou excluir, e daí a partir daí você não vai mais conseguir conectar nesse banco"*.
`m2p3:169`

> **Nomenclatura:** a aula chama de **"token"** ao desenhar (`m2p3:66`) e de **"chave"** na tela
> (`m2p3:78`); os manuais chamam de **"token de pareamento"** (`relay.yaml: token:`). É o mesmo
> objeto — não é divergência, é sinônimo.

---

## 4. Passo 2 — Definir os schemas/tabelas permitidos no Shift

`[UI-OBSERVADA]` `m2p3:81-86`. No relay recém-criado há um **ícone de escudo**, rotulado
**"Tabelas permitidas na borda"**.

`[VÍDEO]` O que se digita e por quê:

> *"Você pode definir quais tabelas a borda vai permitir que o Shift conecte. Isso é pra dar uma
> segurança maior."* — `m2p3:83`

`[UI-OBSERVADA]` Em aula foram digitados os schemas **seguidos de ponto** e salvos:
`VIASOFTMCP.`, `VIASOFTBASE.`, `VIASOFTFIN.` — `m2p3:84-85`. A tela responde com um texto do tipo
*"só essas tabelas podem receber…"* — `m2p3:86`.

> ⚠️ **Inconsistência dentro da própria aula, registrada como está:** a fala do `m2p3:84` diz
> ponto (`VIASOFTMCP.`) e a do `m2p3:86` diz *"como eu coloquei `.*` todas as tabelas desse schema
> vão ser aceitas"*. Os manuais usam **`.*`** sem ambiguidade: *"Padrões separados por vírgula:
> `VIASOFTBASE.*, CONSTRUSHOW.PESSOA`"* — `instalar-o-relay-no-windows.md § 5`.
> **Use `SCHEMA.*`, separado por vírgula.** `[CONFIRMADO-DOC]` vence a fala.

> ⚠️ **`[CONFIRMADO-DOC]` Allowlist vazia = nada permitido.** *"Sem allowlist o relay **nega
> tudo**, por design"* (`instalar-o-relay-no-windows.md § 11`); e
> `instalar-o-relay-no-linux.md § 5`: *"Allowlist vazia ou ausente = nada é permitido (secure by
> default): o destino com credencial mas sem allowlist recusa toda carga/leitura."*

> **D-R5 — a allowlist aparece em DOIS lugares e ninguém explica a relação.**
> - `[UI-OBSERVADA]` Tela do Shift, no relay: escudo "Tabelas permitidas na borda". `m2p3:82-86`
> - `[UI-OBSERVADA]` **Instalador**, no cadastro de cada banco: campo "Allowed tables" / tabelas
>   permitidas — o instrutor redigita os mesmos schemas. *"Aqui a mesma coisa, quais que são as
>   tabelas lá que você autorizou"* — `m2p3:121`
> - `[CONFIRMADO-DOC]` Os manuais descrevem **apenas a local**, no `relay.yaml`
>   (`apply_allowlist` / `extract_allowlist`), e **não mencionam a tela do Shift**.
>
> **`[LACUNA]`** Falta descobrir: a tela do Shift **empurra** a allowlist para o relay, ou são
> duas listas independentes que precisam bater à mão? Se forem independentes, divergência entre
> elas é uma fonte de falha silenciosa. A aula sugere independência (redigitou), mas
> **não afirma**.

> **Nota de escopo:** `[CONFIRMADO-DOC]` a allowlist de tabelas só é **obrigatória quando se
> informa usuário** no destino, isto é, quando o destino vai **aplicar/extrair na borda**
> (Oracle). Destino sem credencial é *"apenas de túnel"* e não usa allowlist de tabela — mas
> continua limitado à allowlist de `host:porta`. — `instalar-o-relay-no-windows.md § 5`

`[UI-OBSERVADA]` Enquanto o serviço não subir, o relay aparece **OFFLINE** na lista. `m2p3:87`

---

## 5. Passo 3 — Baixar o instalador

`[UI-OBSERVADA]` `m2p3:95-103`. Na tela **Relays** há o botão **`Baixar instalador`**, presente
tanto em Configurações (espaço) como no grupo econômico (`m2p3:96-97`). O arquivo é
**`shift-relay-installer.exe`** (`m2p3:103`), coerente com `instalar-o-relay-no-windows.md § 4`
`[CONFIRMADO-DOC]`.

> ⚠️ **O botão só aparece se a Viasoft publicou uma versão.** `[UI-OBSERVADA]` Em aula o botão
> **não estava lá** e o instrutor precisou publicar antes: *"não tá aparecendo porque eu não tinha
> subido o instalador aqui"* → **Substituir Instalador** → escolher um arquivo compactado →
> **Publicar**. `m2p3:90-94`
> `[CONFIRMADO-DOC]` Bate com o manual: *"disponível para ADMIN, **quando a Viasoft publicou uma
> versão**"* — `§ 4`.

`[CONFIRMADO-DOC]` Sem download na plataforma, o instalador é gerado no repositório com
`.\scripts\build-relay-installer.ps1 -Version 1.2.3` (saída em
`shift-connector\dist\shift-relay-installer.exe`). — `§ 4`

`[UI-OBSERVADA]` **SmartScreen bloqueia.** Em aula: "Executar mesmo assim" (`m2p3:104-105`).
`[CONFIRMADO-DOC]` Confirmado e explicado: *"enquanto o binário não for assinado, clique em Mais
informações → Executar assim mesmo. Se o EDR/AV do cliente bloquear, crie exclusão para a pasta de
instalação."* — `§ 5`

`[UI-OBSERVADA]` A versão instalada em aula era a **1.0** ("este instalará o Shift Relay versão
1.0"). `m2p3:106`

---

## 6. Passo 4 — Rodar o instalador (Windows)

`[CONFIRMADO-DOC]` Executar **como Administrador** (o wizard pede elevação). Quatro paradas.
— `instalar-o-relay-no-windows.md § 5`

### Ordem das telas

> **D-R4 — a ordem das telas divergem entre aula e manual.**
>
> | # | `[CONFIRMADO-DOC]` manual `§ 5` | `[UI-OBSERVADA]` aula |
> |---|---|---|
> | 1 | **Local de destino** — pasta do programa, padrão `C:\Program Files\ShiftRelay` | **Token** + **Nome** (`m2p3:108-111`) |
> | 2 | **Token do relay** (nome opcional) | **"onde ele vai salvar"** — `c:\ProgramData\Shift\Relay` (`m2p3:112`) |
> | 3 | **Pasta de dados**, padrão `C:\ProgramData\Shift\relay` | **Bancos de dados** (`m2p3:114-123`) |
> | 4 | **Bancos de dados** | **"onde ele vai salvar outras informações"** (`m2p3:124`) |
>
> `[INFERIDO]` A leitura mais provável: a aula chamou a **pasta de dados** de "onde ele vai
> salvar", e a tela de "outras informações" do passo 4 é a pasta do programa — a fala não
> distingue as duas pastas. **Sustenta a inferência:** no `m2p3:159-162` o instrutor vai buscar o
> `relay.yaml` em **`C:\Program Files\ShiftRelay`**, exatamente onde o manual diz que ele é
> gravado (`§ 5`, fim). Registrado como divergência porque **não está provado** — a ordem real das
> telas precisa ser conferida no instalador.

### As duas pastas — não confundir

`[CONFIRMADO-DOC]` `instalar-o-relay-no-windows.md § 5` e `§ 9`:

| Pasta | Padrão | Contém | No update/uninstall |
|---|---|---|---|
| **Programa** | `C:\Program Files\ShiftRelay` | `shift-relay.exe` + **`relay.yaml`** | apagada no desinstalar |
| **Dados** | `C:\ProgramData\Shift\relay` | logs, estado dos jobs, arquivos da borda, **watermark das capturas** | **preservada** — apagar faria reprocessar tudo |

> `[CONFIRMADO-DOC]` *"Prefira um disco com espaço"* na pasta de dados se o cliente vai usar
> aplicação/extração na borda. — `§ 5`

### Campos do cadastro de cada banco

`[CONFIRMADO-DOC]` `§ 5`, cruzado com o que a aula preencheu `[UI-OBSERVADA]` `m2p3:114-123`:

| Campo (manual) | Rótulo/valor na aula | O que informar |
|---|---|---|
| **Nome** | `Local` (`m2p3:116`) | Apelido do banco (ex.: `oracle-erp`). É o **alias usado pelos jobs da borda** |
| **Host do banco** | `192.168.91.89` (`m2p3:114-116`) | Como **esta máquina** vê o banco (`localhost`, IP ou hostname da LAN) |
| **Porta do banco** | `1521` (`m2p3:117`) | 1521 Oracle · *(1433 SQL Server, 5432 Postgres, 3050 Firebird — fora de escopo)* |
| **Número do destino** (`remote_port`) | **"Porta do gateway"** = `11521` (`m2p3:117-119`) | **Ver D-R1 e D-R2** |
| **Service name** | `VIASOFT3` (`m2p3:120`) | **Só** se o destino vai aplicar/extrair na borda (Oracle) |
| **Usuário / senha** | `VIASOFTMCP` (`m2p3:120`) | idem |
| **Tabelas permitidas** | `VIASOFTBASE.`, `VIASOFTMCP.`, `VIASOFTFIN.*` (`m2p3:121`) | Obrigatório quando informa usuário |

`[UI-OBSERVADA]` Depois de preencher, botão **"Adicionar banco"** (a aula lê "Adicionar banco
cadastrado") e o banco vai para uma lista; então **Avançar** → **Instalar** → **Concluir**.
`m2p3:122-127`

> **D-R1 — o nome do campo da porta. Divergência de vocabulário com consequência prática.**
> | Fonte | Como chama |
> |---|---|
> | `[UI-OBSERVADA]` instalador, aula | **"Porta do gateway"** (`m2p3:117-118`) |
> | `[UI-OBSERVADA]` conexão no Shift, aula | **"Porta no gateway"** (`m2p3:154-155`) |
> | `[UI-OBSERVADA]` `relay.yaml` na aula | **`gateway_port: 18521`** (`m2p3:162`) |
> | `[CONFIRMADO-DOC]` manuais | **"Número do destino"** / **"Porta do destino no relay"** / chave YAML **`remote_port`** |
>
> **`gateway_port` não existe em nenhum documento** — varredura em `bruto/docs/**/*.md`: zero
> ocorrências; `remote_port` aparece 12 vezes nos dois manuais. **`[LACUNA]`** Falta descobrir se
> a chave do YAML mudou de nome entre a versão 1.0 da aula e a documentada, ou se as duas chaves
> coexistem. **Consequência:** ao editar um `relay.yaml` à mão, conferir qual chave o binário
> instalado aceita — errar isso derruba o serviço com `config inválida`.

> **D-R2 — que número usar na porta. Duas convenções incompatíveis.**
> - `[VÍDEO]` **Aula: prefixar `1` na porta do banco.** *"É importante que você sempre use uma
>   porta aleatória […] obviamente uma porta não utilizada. Então por convenção geralmente
>   utiliza-se o um na frente. Então aqui no caso a porta é 1521, vou botar 11521."* — `m2p3:118-119`
> - `[CONFIRMADO-DOC]` **Manual: o instalador já sugere `15432`** e o próximo livre nos bancos
>   seguintes; no Linux, *"comece em `15432` e siga (`15433`, …)"*.
>   — `instalar-o-relay-no-windows.md § 2`; `instalar-o-relay-no-linux.md § 2`
>
> `[CONFIRMADO-DOC]` **O que os dois concordam:** o número **vale só dentro desta instalação**,
> precisa ser **diferente entre os bancos da mesma instalação**, e **repetir o mesmo número em
> outros clientes é normal**. E: *"Nada sobre portas do lado do Shift […] O endereço do lado do
> Shift é gerado por ele; não há o que digitar."*
>
> `[INFERIDO]` A convenção "1 na frente" é heurística pessoal do instrutor, não regra da
> plataforma — o campo já vem preenchido pelo instalador.

> **D-R3 — o nome do relay é obrigatório ser igual, ou opcional?**
> - `[VÍDEO]` **Aula: obrigatório e idêntico.** *"É bom sempre usar o mesmo nome. […] não é bom,
>   é **é obrigatório** que seja o mesmo, senão não vai conseguir se encontrar."* — `m2p3:144`
> - `[CONFIRMADO-DOC]` **Manual: opcional.** *"O **nome** é opcional (vazio vira `relay`); use o
>   mesmo nome cadastrado no Shift para facilitar o suporte."* — `§ 5`
> - `[CONFIRMADO-DOC]` **Mas a tabela de problemas do mesmo manual dá razão à aula:** sintoma
>   *"Conecta e cai em seguida"* → causa *"Token revogado/errado, ou **nome do relay diferente do
>   cadastrado no Shift**"* — `§ 11`. Idem no Linux: *"`relay_name` diferente do cadastrado no
>   Shift"* (`§ 11`).
>
> **O manual se contradiz internamente.** `[INFERIDO]` Na dúvida, **use exatamente o mesmo nome**
> — é o comportamento seguro nas três fontes. **`[LACUNA]`** Falta descobrir se o `relay_name` é
> chave de pareamento junto com o token, ou só rótulo de log.

---

## 7. Passo 4 (Linux) — resumo

`[CONFIRMADO-DOC]` `instalar-o-relay-no-linux.md § 3-7`. Sem wizard: o binário é o instalador.

```
install -m 0755 shift-relay-linux-amd64 /usr/local/bin/shift-relay
mkdir -p /etc/shift-relay /var/lib/shift-relay
# escrever /etc/shift-relay/relay.yaml  →  chmod 600 (não é opcional: guarda token e senhas)
shift-relay check   -config /etc/shift-relay/relay.yaml
shift-relay install -config /etc/shift-relay/relay.yaml
shift-relay start
```

| Caminho | Papel |
|---|---|
| `/usr/local/bin/shift-relay` | binário |
| `/etc/shift-relay/relay.yaml` | configuração (`chmod 600`) |
| `/var/lib/shift-relay` | `data_dir` — **fora do caminho do binário de propósito**, para sobreviver a atualizações |

Pontos que só existem no Linux `[CONFIRMADO-DOC]`:
- `install` grava `/etc/systemd/system/ViasoftShiftServiceRelay.service`, roda `systemctl enable`
  e `daemon-reload`. **`Restart=always` com `RestartSec=120`** — 2 minutos para voltar após queda,
  contra 10s do SCM no Windows.
- `install` **falha com `Init already exists`** se a unit já existir → rodar `uninstall` antes.
- A unit gerada roda como **root**. Para usuário dedicado, **não** usar `shift-relay install`:
  escrever a unit à mão (`§ 7`, variante documentada com `NoNewPrivileges`, `ProtectSystem=strict`,
  `ReadWritePaths`).
- Variante **container** documentada (`docker run --network host`, logs em stdout).
- Em SELinux *enforcing*, `/usr/local/bin` já tem contexto correto; outro caminho exige
  `restorecon -v`.

---

## 8. Passo 5 — Conferir que subiu e ficou ONLINE

`[UI-OBSERVADA]` `m2p3:128-136`:

1. Abrir **Serviços** do Windows → letra S → **`Shift Relay`**, status *"Em Execução"*.
2. Voltar ao Shift e **atualizar a página** → o relay muda de **OFFLINE** para **ONLINE**.
3. `[VÍDEO]` *"Ou seja, o túnel de conexão entre o Shift e a minha máquina foi estabelecido. **Mas
   esse é só o túnel, a conexão ainda não foi criada.**"* — `m2p3:136-137`

`[CONFIRMADO-DOC]` O comando canônico e os nomes reais — `instalar-o-relay-no-windows.md § 6`:

```
Get-Service ViasoftShiftServiceRelay          # deve dizer Running
Get-Content 'C:\ProgramData\Shift\relay\logs\shift-relay.log' -Tail 40
```

| O que | Nome |
|---|---|
| Nome do serviço | **`ViasoftShiftServiceRelay`** |
| Nome de exibição (o que aparece no `services.msc`) | **Shift Relay** |

`[CONFIRMADO-DOC]` Ciclo de vida (start/stop/panic) vai para **Visualizador de Eventos →
Aplicativo**, origem `ViasoftShiftServiceRelay`; o log detalhado do túnel vai para o arquivo.
Do lado do Shift, a tela **Relays** mostra o relay ativo **com a versão do binário anunciada no
handshake**.

`[INFERIDO]` **Como o Shift decide "ONLINE":** `extracao-na-borda-design.md § Matriz de
elegibilidade` cita `tunnel_relays.last_seen_at` dentro de `RELAY_ONLINE_WINDOW_SECONDS` (**90s**).
Marcado `[INFERIDO]` por regra da casa — é documento de design. Não confirmado como o mecanismo da
tela.

---

## 9. Passo 6 — Apontar a conexão para o relay

`[UI-OBSERVADA]` `m2p3:137-155`. É uma conexão normal (ver `configurar-conexao-direta.md`) com
**três diferenças**:

| # | Campo | Valor em aula | Linha |
|---|---|---|---|
| 1 | Nome | `Local` — **o mesmo nome do relay** | `m2p3:138` |
| 2 | Sistema | `CONSTRUSHOW` | `m2p3:139-146` |
| 3 | Tipo | Oracle | `m2p3:145` |
| 4 | **Formato** | **Easy Connect** | `m2p3:149` |
| 5 | Host | `192.168.91.89` | `m2p3:147-148` |
| 6 | Banco | `VIASOFT3` | `m2p3:150-151` |
| 7 | Schemas | `VIASOFTMCP`, `VIASOFTBASE`, `VIASOFTFIN` | `m2p3:150-151` |
| 8 | ☑ **"Conexão de relay (banco on-premise)"** + escolher o relay `Local` | marcado | `m2p3:152-153` |
| 9 | **"Porta no gateway"** | `11521` → depois corrigido para `18521` | `m2p3:154-165` |

> `[VÍDEO]` **Com relay, use Easy Connect.** *"Quando é por relay você pode sempre deixar easy
> connect aqui, quando é por aplicação de borda."* — `m2p3:149`

> `[VÍDEO]` **É obrigatório ter um Sistema cadastrado** antes — o instrutor teve de sair e criar o
> sistema `CONSTRUSHOW` (Oracle) nas configurações do espaço. *"É obrigatório ter um sistema."* —
> `m2p3:140-141`. Mesmo pré-requisito da conexão direta.

`[CONFIRMADO-DOC]` O manual descreve os mesmos 3 passos e explica o efeito:
*"Host e porta que o Shift usa passam a ser os do **gateway**; o relay repassa para o banco real na
LAN do cliente."* — `instalar-o-relay-no-windows.md § 7`

> **`[LACUNA]` — contradição aparente que a aula não explica.** O manual diz que *"o endereço do
> lado do Shift é gerado por ele; não há o que digitar"* (`§ 7`), mas a aula **digita host
> `192.168.91.89` e banco `VIASOFT3` na conexão** (`m2p3:147-151`). Falta descobrir para que o
> Shift usa esse host quando a conexão é por relay: é ignorado? é o que o relay recebe como
> destino? é só metadado? Isso é **exatamente o tipo de campo que faz a conexão falhar sem
> mensagem clara**.

---

## 10. A armadilha de porta em uso — o único erro real da aula

`[UI-OBSERVADA]` `m2p3:156-166`. Ao clicar **Criar**, a conexão **falhou**. Reconstituição do que
o instrutor relatou depois da pausa de gravação:

1. A porta escolhida **já estava em uso por outra instalação** na mesma máquina. `m2p3:158, 161`
2. Correção aplicada: **parar o serviço Shift Relay** (Serviços do Windows) → editar o
   **`relay.yaml`** → trocar a porta → subir de novo. `m2p3:159-162`
3. Onde está o arquivo: *"por padrão sempre vai ser Disco C, Arquivos de Programas, Shift Relay"* →
   `C:\Program Files\ShiftRelay\relay.yaml`. `m2p3:159-160`
4. Linha alterada, lida na tela: **`gateway_port: 18521`**. `m2p3:162`
5. De volta ao Shift: trocar **"Porta no gateway"** para `18521` → **Salvar** → **Testar conexão**
   → *"conexão bem-sucedida"*. `m2p3:164-166`

> ⚠️ **Falha de usabilidade declarada pelo próprio instrutor:** *"vou ter que só melhorar a
> mensagem ali pra ele gerar uma nova porta […] em produção eu vou ajustar pra essa mensagem ficar
> mais clara, ele gerar algumas portas de sugestão pra vocês usarem."* — `m2p3:158, 163`
> **Ou seja: a mensagem de erro de porta ocupada não diz o que fazer.** Suspeite dela primeiro.

> ⚠️ **Inconsistência numérica dentro da transcrição, registrada sem escolha:** `m2p3:119` diz
> `11521`; `m2p3:158` diz *"antes tava aqui **15521**"*; o valor final é `18521`
> (`m2p3:161-162, 165`). Provável lapso de fala, mas **não há como decidir qual número foi
> digitado** — não use nenhum dos três como referência.

`[CONFIRMADO-DOC]` O manual cobre o mesmo sintoma por dois ângulos —
`instalar-o-relay-no-windows.md § 11`:

| Sintoma | Causa |
|---|---|
| `remote_port já usada` | Dois destinos com o mesmo número **nesta** instalação — dentro do relay o número é a chave do destino |
| Conexão do Shift falha, **mas o relay está online** | Número do destino no cadastro da conexão **≠** `remote_port` do relay; ou o banco não aceita conexão vinda da máquina do relay |

`[INFERIDO]` O caso da aula é o **segundo** (relay ONLINE, conexão falha), não o primeiro — a
colisão foi com outra instalação na mesma máquina, não entre destinos do mesmo relay. Mas o
manual diz que a colisão só importa *dentro* da instalação, o que **não explica** o erro da aula.
**`[LACUNA]`**: falta descobrir se a porta é ocupada no SO da máquina do relay (o que
contradiria "o relay não escuta porta", `instalar-o-relay-no-linux.md § 2`) ou se a colisão foi no
gateway.

---

## 11. Operação do dia a dia

`[CONFIRMADO-DOC]` `instalar-o-relay-no-windows.md § 8`:

```
& 'C:\Program Files\ShiftRelay\shift-relay.exe' status
& 'C:\Program Files\ShiftRelay\shift-relay.exe' stop
& 'C:\Program Files\ShiftRelay\shift-relay.exe' start
& 'C:\Program Files\ShiftRelay\shift-relay.exe' version
& 'C:\Program Files\ShiftRelay\shift-relay.exe' check -config 'C:\Program Files\ShiftRelay\relay.yaml'
```

- **Sobe no boot** e o SCM **reinicia em 10s** se o processo cair (Linux: 120s).
- **Logs rotacionados** em `<pasta de dados>\logs\shift-relay.log`: 20 MB por arquivo, 5 backups,
  30 dias (teto ~100 MB).
- **Nunca loga credencial nem linha de dado.**

### Bundle de diagnóstico — o que pedir ao TI do cliente

`[CONFIRMADO-DOC]` Um comando só, em vez de sessão remota:

```
& 'C:\Program Files\ShiftRelay\shift-relay.exe' diag -config 'C:\Program Files\ShiftRelay\relay.yaml'
```

Gera `shift-relay-diag-<timestamp>.zip` no diretório atual com logs, o `relay.yaml`
**sanitizado** (token/senhas viram `***`), snapshot de versão/capabilities/destinos e um teste de
conectividade (**DNS, TCP no gateway, skew de relógio**). Funciona com o serviço **rodando ou
parado** e **não contém credencial nem linha de dado**. — `§ 8`

### Atualizar e desinstalar (Windows)

`[CONFIRMADO-DOC]` `§ 9`:
- **Atualizar:** rodar o instalador novo **por cima** — é **idempotente**. Para o serviço, troca o
  `.exe`, **pré-carrega** token, nome, pasta de dados, destinos e allowlist do config existente,
  reescreve o `relay.yaml`, reinstala e reinicia. **Pasta de dados preservada** (offsets intactos →
  nada é reprocessado). **Não há auto-update:** a janela de atualização é do cliente.
- **Desinstalar:** *Configurações → Aplicativos → Shift Relay → Desinstalar*. Remove serviço,
  `relay.yaml` e `.exe`. **A pasta de dados NÃO é removida.**

> ⚠️ `[CONFIRMADO-DOC]` **Instalação silenciosa não é suportada.** Com `/VERYSILENT` o wizard não
> coleta token nem destinos e *"o `relay.yaml` sai inválido"*. Para automação, usar a instalação
> manual (`§ 10`: copiar `.exe` + `relay.yaml`, então `shift-relay install` e `start`) ou o bundle
> `shift-relay-windows-amd64.zip` com `instalar.bat` + `setup.ps1`. — `§ 5` e `§ 10`

---

## 12. Referência do `relay.yaml`

`[CONFIRMADO-DOC]` `instalar-o-relay-no-windows.md § 12` (versão Windows; a Linux é idêntica com
caminhos POSIX):

```yaml
gateway_url: wss://tunnel-shift.viasoftcloud.com.br/tunnel
relay_name: erp-matriz          # = nome do relay no Shift
token: <token de pareamento>
fingerprint: ""                 # sha256 do cert do gateway (pin opcional)
tls_max_version: ""             # vazio = 1.3 com fallback automático p/ 1.2
log_level: info                 # info | debug (debug é volumoso)
keepalive_seconds: 25
data_dir: 'C:\ProgramData\Shift\relay'

destinations:
  - name: oracle-erp            # alias usado pelos jobs da borda
    remote_port: 15432          # número do destino nesta instalação
    host: 10.0.0.30             # como ESTA máquina vê o banco
    port: 1521
    # --- opcional: aplicar/extrair na borda (Oracle) ---
    apply_dialect: oracle
    apply_service: ORCLPDB1
    apply_user: SHIFT_LOAD
    apply_password: '<senha>'
    apply_allowlist:
      - "VIASOFTBASE.*"
```

`[CONFIRMADO-DOC]` No Linux existem também as chaves de **leitura** na borda —
`extract_dialect` / `extract_service` / `extract_user` / `extract_password` / `extract_allowlist` —
mostradas em `instalar-o-relay-no-linux.md § 5`. **A referência do Windows não as lista.**
`[INFERIDO]` São do mesmo binário, portanto válidas nos dois; registrado porque é assimetria de
documentação, não de produto.

Três cuidados declarados `[CONFIRMADO-DOC]`:
1. `data_dir` **entre aspas simples** no Windows: YAML single-quoted preserva as `\`; aspas duplas
   interpretariam `\r`, `\P` como escape.
2. O `relay.yaml` **guarda token e senhas** — restringir a permissão do arquivo à conta do serviço
   (Linux: `chmod 600`).
3. Campos obrigatórios cuja ausência derruba o serviço com `config inválida`: **`gateway_url`,
   `relay_name`, `token`**, e destino sem `host`/portas. Valide com `shift-relay check`.

> ⚠️ **Ver D-R1:** a aula lê **`gateway_port`** no arquivo real da versão 1.0; a documentação usa
> **`remote_port`**. Antes de editar à mão, confira qual chave o binário instalado aceita.

---

## 13. Extração e aplicação na borda — estado atual

`[CONFIRMADO-MCP]` O nó **`sql_database`** (Extração SQL, `input`, risco `read_only`) **já tem o
campo**:

> `read_delivery_mode` (enum: `auto`, `tunnel`, `edge`) — *"Modo de leitura para conexões via
> relay: auto (decide borda vs túnel), tunnel (sempre túnel) ou edge (sempre na borda)"*

Ou seja, **a superfície de configuração existe no contrato do nó, hoje.**

`[INFERIDO]` — de `extracao-na-borda-design.md` (documento de **design**, não pode ser
`[CONFIRMADO-DOC]` por regra da casa):
- A funcionalidade é regida pela flag **`RELAY_EXTRACT_ENABLED`, default `False`**; enquanto
  `False`, *"TODA leitura vai pelo túnel"* e o roteamento **nem é avaliado**.
- Limiar **`RELAY_EXTRACT_MIN_ROWS = 50_000`** (escrita: `RELAY_APPLY_MIN_ROWS = 10_000`).
- Elegibilidade exige: conexão via relay + origem **Oracle** + relay anunciando capability
  `extract:oracle` + relay **ONLINE** (janela de 90s) + volume estimado ≥ limiar.
- Em `auto`, qualquer inelegibilidade cai para o túnel; **`edge` explícito com inelegibilidade dá
  erro** (`RelayExtractForcedButIneligible`), sem fallback silencioso.
- O documento se declara **inerte até a T4** (CIAG-271).

> ⚠️ **Consequência de leitura:** encontrar `read_delivery_mode` na tela **não significa** que a
> extração na borda esteja ligada. **`[LACUNA]`** Não há como verificar pelo MCP o valor de
> `RELAY_EXTRACT_ENABLED` neste ambiente, nem as `capabilities` anunciadas por um relay.

---

## 14. O que o MCP confirma sobre relay

`[CONFIRMADO-MCP]` `diagnosticar_conexao` **é ciente de relay** por contrato — a descrição da
ferramenta diz que ela diagnostica *"levando em conta se a conexão passa por relay (o mesmo erro
tem causa diferente com e sem ponte)"* e que **"se a conexão usa relay e ele está offline, responde
isso primeiro, sem investigar o banco"**.

`[CONFIRMADO-MCP]` Verificado nas duas conexões Oracle reais deste ambiente: campo
**`via_relay: false`** nas duas, com `dns`/`tcp`/`auth_query` ok.

`[CONFIRMADO-MCP]` **Não existe nenhuma ferramenta de relay no MCP** — nada como `list_relays`,
`get_relay` ou `relay_status`. O relay só é observável indiretamente, pelo `via_relay` do
diagnóstico.

> ⚠️ `[CONFIRMADO-MCP]` `diagnosticar_conexao` **não serve para cadastro ainda não salvo** — *"a
> senha do banco não trafega na conversa"*. Nesse caso, pedir para a pessoa usar o botão **Testar
> conexão** da tela e colar o resultado. Isso importa: durante a instalação de um relay, o que
> falha normalmente é **antes** de a conexão estar salva.

---

## 15. Divergências deste procedimento — resumo

| # | Tema | Aula | Doc | Estado |
|---|---|---|---|---|
| **D-R1** | Nome da porta | "Porta do gateway/no gateway"; YAML **`gateway_port`** (`m2p3:118, 154, 162`) | "Número do destino"; YAML **`remote_port`** (`§ 12`) | Aberta. `gateway_port` tem **0 ocorrências** nos docs |
| **D-R2** | Valor da porta | prefixar `1` na porta do banco → `11521` (`m2p3:118-119`) | instalador **sugere `15432`** e incrementa (`§ 2`) | Aberta; provável heurística pessoal |
| **D-R3** | Nome do relay | **"é obrigatório que seja o mesmo"** (`m2p3:144`) | **"opcional (vazio vira `relay`)"** (`§ 5`) — mas `§ 11` culpa o nome divergente por queda | Doc **se contradiz**; usar o mesmo nome |
| **D-R4** | Ordem das telas do wizard | Token → pasta → bancos → pasta (`m2p3:108-124`) | Local de destino → Token → Pasta de dados → Bancos (`§ 5`) | Aberta; provável imprecisão da fala |
| **D-R5** | Onde vive a allowlist | tela do Shift (escudo) **e** instalador (`m2p3:82-86, 121`) | **só** local, no `relay.yaml` (`§ 5`, `§ 12`) | Aberta — falta saber se sincronizam |
| **D-R6** | Linux existe? | *"por enquanto só Windows"* (`m2p3:99`) | manual Linux completo | Aula anterior ao doc; **sem wizard no Linux** |
| **D-R7** | Onde criar o relay | menu **Relays no grupo econômico** (`m2p3:71`) e em Configurações (`m2p3:90`) | só **"Configurações → Relays"** (`§ 3`) | Aberta |

---

## 16. Observações para o piloto

1. **O relay não é necessário para o piloto de margem.** `[CONFIRMADO-MCP]` As duas conexões
   Oracle do ambiente (`192.168.90.218`, portas 30200/30100, banco `ORCL`, usuário `VIASOFTMERC`)
   respondem `status: ok` e **`via_relay: false`**. O caminho de rede já existe sem ponte.
2. **Onde ele voltaria a importar:** (a) se o piloto for replicado em cliente on-premise, e (b)
   se a **extração na borda** for ligada para acelerar leituras grandes de pedidos — e isso
   depende de o relay existir, porque o roteamento de borda exige `connections.relay_id`
   preenchido. Hoje, com `via_relay: false`, **as conexões do piloto são estruturalmente
   inelegíveis para extração na borda**, independentemente da flag.
3. **Se um dia entrar relay:** o campo `read_delivery_mode` do nó `sql_database` já existe
   `[CONFIRMADO-MCP]` — mas deixar em `auto` é o único valor seguro, porque `edge` com
   inelegibilidade **falha em vez de cair para o túnel** `[INFERIDO]`.
4. **O risco operacional mais próximo do piloto não é técnico, é de escopo:** a orientação de aula
   de **nunca** pôr conexão quente de cliente no nível do espaço (`m2p3:170-173`). As duas conexões
   Oracle deste ambiente têm dono **workspace** e são **públicas** (`get_connection`:
   `Workspace: fd9cff30…`, `Publica: Sim`) — exatamente o arranjo que a aula recomenda evitar para
   banco quente. Ver `glossario.md § Conexão` e a lacuna de `environment` não retornado pelo MCP.
