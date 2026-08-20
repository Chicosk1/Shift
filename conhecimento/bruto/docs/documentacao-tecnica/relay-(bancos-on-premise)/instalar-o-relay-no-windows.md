---
title: Instalar o Relay no Windows
---

Manual de instalação do `shift-relay` — o componente que roda **na infra do
cliente** e permite que o Shift (nuvem) alcance bancos de dados on-premise **sem
abrir nenhuma porta de entrada no firewall**.

Para servidores Linux, ver [Instalar o Relay no Linux](https://shift.viasoftcloud.com.br/docs/technical/relay/instalacao-linux).

---

## 1. O que o relay faz

```
  Rede do cliente                              Nuvem
  ┌──────────────┐     ┌─────────────┐        ┌───────────────┐
  │ banco(s)     │◄────│ shift-relay │ ──wss─►│ shift-gateway │◄── Shift
  │ ERP/Oracle/… │ LAN │  (serviço)  │ saída  │               │
  └──────────────┘     └─────────────┘  443   └───────────────┘
```

- **Só tráfego de saída.** O relay sempre inicia a conexão; o gateway nunca
alcança a rede do cliente por conta própria.

- **Burro de propósito.** Não conhece tipo de banco, não tem driver instalado,
não vê usuário/senha/SQL do fluxo — para ele é tudo TCP. A definição da
conexão vive no Shift.

- **Allowlist local.** O relay só alcança os `host:porta` declarados no config
dele. Não é um proxy genérico.

- **Um binário, sem dependências.** Go estático: sem JVM, sem .NET, sem Oracle
Instant Client.

> Se origem e destino estão em redes isoladas (filiais/VLANs sem rota),
> instala-se **um relay por rede**. Bancos na mesma LAN são atendidos por um
> relay só — basta listar os dois destinos.

---

## 2. Antes de começar

### Na máquina do cliente

| Item | Requisito |
| --- | --- |
| Sistema | Windows x64 (Windows 10/11 ou Windows Server 2016+) |
| Permissão | Conta **Administrador local** (o instalador registra um serviço) |
| Disco | ~100 MB para o programa + espaço na **pasta de dados** (logs, e os Parquets da borda, que podem ser grandes) |
| Rede | Saída HTTPS/443 liberada para o endereço do gateway (`tunnel-shift.viasoftcloud.com.br`) |
| Relógio | Sincronizado (NTP). Skew grande atrapalha TLS e é sinalizado pelo `diag` |

> **Proxy corporativo:** o relay **não** usa as variáveis `HTTP_PROXY`/
> `HTTPS_PROXY` para a conexão `wss`. Em rede com proxy explícito obrigatório,
> libere saída direta na 443 para o host do gateway.

### Informações que a Viasoft/o operador do Shift precisa fornecer

- **Token do relay** (gerado no Shift, exibido **uma única vez**).

- **Nome do relay** (identifica a máquina nos logs e na tela de relays).

- Nada sobre portas do lado do Shift: o **número do destino** (`remote_port`) é
escolhido aqui, vale só dentro desta instalação e o instalador já sugere
`15432` (o próximo livre nos bancos seguintes). É esse mesmo número que se
informa depois no cadastro da conexão, no Shift.

- Se o destino também vai **aplicar/extrair dados na borda** (Oracle):
service name, usuário, senha e a lista de tabelas permitidas.

---

## 3. Passo 1 — Criar o relay no Shift e copiar o token

No Shift, com perfil **ADMIN** do workspace/grupo econômico:

1. Abra **Configurações → Relays**.

2. **Novo relay** → dê um nome (ex.: `Servidor ERP — Matriz`) → **Criar**.

3. **Copie o token de pareamento.** Ele é mostrado **uma vez só**; se perder,
crie outro relay (ou revogue e recrie).

---

## 4. Passo 2 — Baixar o instalador

Ainda na tela **Relays**, use **Baixar instalador** (disponível para ADMIN,
quando a Viasoft publicou uma versão). O arquivo é o
`shift-relay-installer.exe`.

Não havendo o download na plataforma, o instalador é gerado no repositório com:

```
.\scripts\build-relay-installer.ps1 -Version 1.2.3
```

(saída em `shift-connector\dist\shift-relay-installer.exe`).

---

## 5. Passo 3 — Rodar o instalador na máquina do cliente

Execute o `shift-relay-installer.exe` **como Administrador** (o wizard pede
elevação). Ele é todo em português e tem quatro paradas:

1. **Local de destino** — pasta do programa. Padrão
`C:\Program Files\ShiftRelay`; pode trocar (ex.: `D:\ShiftRelay`).

2. **Token do relay** — cole o token do passo 1. O **nome** é opcional (vazio
vira `relay`); use o mesmo nome cadastrado no Shift para facilitar o suporte.

3. **Pasta de dados** — onde ficam logs, estado dos jobs, arquivos da borda e o
watermark das capturas. Padrão `C:\ProgramData\Shift\relay`. **Prefira um
disco com espaço** se o cliente vai usar aplicação/extração na borda.

4. **Bancos de dados** — para cada banco, preencha e clique **Adicionar banco**:

| Campo | O que informar |
| --- | --- |
| Nome | Apelido do banco (ex.: `oracle-erp`). É o alias usado pelos jobs da borda |
| Host do banco | Como **esta máquina** enxerga o banco (`localhost`, IP ou hostname da LAN) |
| Porta do banco | 1521 Oracle · 1433 SQL Server · 5432 Postgres · 3050 Firebird |
| Número do destino | Já vem preenchido (`15432`, e o próximo livre nos bancos seguintes). Precisa ser diferente entre os bancos **desta** instalação; repetir o mesmo número em outros clientes é normal |
| Service name / usuário / senha | **Só** se este destino vai aplicar/extrair na borda (Oracle). Vazio = destino apenas de túnel |
| Tabelas permitidas | Obrigatório quando informa usuário. Padrões separados por vírgula: `VIASOFTBASE.*, CONSTRUSHOW.PESSOA` |

Ao concluir, o instalador grava o `relay.yaml` na pasta do programa, registra o
serviço do Windows e o inicia.

> **Aviso do SmartScreen** ("Editor desconhecido"): enquanto o binário não for
> assinado, clique em **Mais informações → Executar assim mesmo**. Se o EDR/AV
> do cliente bloquear, crie exclusão para a pasta de instalação.

> **Instalação silenciosa não é suportada.** Com `/VERYSILENT` o wizard não
> coleta token nem destinos e o `relay.yaml` sai inválido. Para automação, use a
> instalação manual da seção 10.

---

## 6. Passo 4 — Conferir que subiu e conectou

No PowerShell (como Administrador):

```
Get-Service ViasoftShiftServiceRelay
```

Deve aparecer `Running`. O serviço registra-se com o nome
`ViasoftShiftServiceRelay` e nome de exibição **Shift Relay** — é assim que
aparece no `services.msc`.

Log do túnel (o que interessa no diagnóstico):

```
Get-Content 'C:\ProgramData\Shift\relay\logs\shift-relay.log' -Tail 40
```

Uma conexão saudável mostra o destino resolvido e a conexão estabelecida com o
gateway. O **ciclo de vida** do serviço (start/stop/panic) vai também para o
**Visualizador de Eventos → Aplicativo**, origem `ViasoftShiftServiceRelay`.

Do lado do Shift, a tela **Relays** passa a mostrar o relay como ativo, com a
versão do binário anunciada no handshake.

---

## 7. Passo 5 — Apontar a conexão no Shift para o relay

No Shift, ao cadastrar/editar a conexão do banco:

1. Escolha o **relay** na conexão.

2. Informe a **Porta do destino no relay** — exatamente o mesmo número
cadastrado aqui (`remote_port`). O endereço do lado do Shift é gerado por ele;
não há o que digitar.

3. **Testar conexão.**

Host e porta que o Shift usa passam a ser os do gateway; o relay repassa para o
banco real na LAN do cliente.

---

## 8. Operação do dia a dia

```
# controle do serviço (pela pasta de instalação)
& 'C:\Program Files\ShiftRelay\shift-relay.exe' status
& 'C:\Program Files\ShiftRelay\shift-relay.exe' stop
& 'C:\Program Files\ShiftRelay\shift-relay.exe' start

# versão instalada
& 'C:\Program Files\ShiftRelay\shift-relay.exe' version

# validar o config (e, havendo capturas, testar a pasta de dados)
& 'C:\Program Files\ShiftRelay\shift-relay.exe' check -config 'C:\Program Files\ShiftRelay\relay.yaml'
```

- **Sobe no boot** e o SCM o **reinicia em 10s** se o processo cair.

- **Logs rotacionados**: `<pasta de dados>\logs\shift-relay.log`, 20 MB por
arquivo, 5 backups, 30 dias (teto ~100 MB).

- **Nunca** loga credencial nem linha de dado.

### Bundle de diagnóstico para o suporte

Em vez de sessão remota, peça ao TI do cliente **um** comando:

```
& 'C:\Program Files\ShiftRelay\shift-relay.exe' diag -config 'C:\Program Files\ShiftRelay\relay.yaml'
```

Gera `shift-relay-diag-<timestamp>.zip` no diretório atual com logs, o
`relay.yaml` **sanitizado** (token/senhas viram `***`), snapshot de versão/
capabilities/destinos e um teste de conectividade (DNS, TCP no gateway, skew de
relógio). Funciona com o serviço **rodando ou parado**.

---

## 9. Atualizar e desinstalar

**Atualizar:** rode o instalador novo **por cima** — ele é idempotente. Para o
serviço, troca o `.exe`, **pré-carrega** token, nome, pasta de dados, destinos e
allowlist do config existente, reescreve o `relay.yaml` e reinstala/reinicia o
serviço. A **pasta de dados é preservada** (offsets de captura não são
apagados, então nada é reprocessado). Não há auto-update: a janela de
atualização é do cliente.

**Desinstalar:** *Configurações → Aplicativos → Shift Relay → Desinstalar*. O
desinstalador para e remove o serviço e apaga `relay.yaml` e o `.exe`. A
**pasta de dados não é removida** — apague à mão se realmente quiser descartar
logs e offsets.

---

## 10. Instalação manual (sem o wizard)

Útil para automação, para máquinas em que só se pode copiar arquivos, ou para
trocar só o binário:

```
# 1. copie shift-relay.exe e um relay.yaml preenchido para a pasta escolhida
# 2. registre e inicie o serviço (PowerShell como Administrador)
& 'C:\Program Files\ShiftRelay\shift-relay.exe' install -config 'C:\Program Files\ShiftRelay\relay.yaml'
& 'C:\Program Files\ShiftRelay\shift-relay.exe' start
```

Para **trocar apenas o binário** numa atualização pontual: `stop` → substituir o
`.exe` → `start` (a configuração e os offsets ficam intactos).

Existe ainda o **bundle .zip** (`shift-relay-windows-amd64.zip`, gerado por
`scripts\build-relay-bundle.ps1`), com `instalar.bat` + `setup.ps1`
perguntando token e bancos no console — alternativa ao wizard quando não se quer
distribuir um `Setup.exe`.

---

## 11. Problemas comuns

| Sintoma | Causa provável / o que fazer |
| --- | --- |
| Serviço não inicia; log diz `config inválida` | Campo obrigatório faltando (`gateway_url`, `relay_name`, `token`) ou destino sem `host`/portas. Rode `shift-relay check -config …` |
| `Get-Service` não encontra o serviço | Procurando pelo nome errado: é `ViasoftShiftServiceRelay` (exibição "Shift Relay") |
| Log repete erro conectando no gateway | Saída 443 bloqueada, DNS não resolve o host do gateway, ou proxy explícito obrigatório (não suportado). Confirme com o `diag` |
| Conecta e cai em seguida | Token revogado/errado, ou nome do relay diferente do cadastrado no Shift |
| `remote_port já usada` | Dois destinos com o mesmo número **nesta** instalação — dentro do relay o número é a chave do destino, então cada banco precisa do seu |
| Conexão do Shift falha, mas o relay está online | Número do destino no cadastro da conexão ≠ `remote_port` do relay; ou o banco não aceita conexão de rede vinda da máquina do relay |
| Job da borda falha com "tabela não permitida" | A allowlist do destino está vazia ou não cobre a tabela. Sem allowlist o relay **nega tudo**, por design |
| SmartScreen/EDR bloqueia o instalador | Binário ainda não assinado — "Executar assim mesmo" ou exclusão para a pasta de instalação |
| Handshake TLS falha em Windows Server antigo | O relay já tenta 1.3 e cai sozinho para 1.2. Para forçar, `tls_max_version: "1.2"` no `relay.yaml` |

---

## 12. Referência rápida do `relay.yaml`

```
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

- `data_dir` entre **aspas simples**: YAML single-quoted preserva as `\` do
caminho Windows (aspas duplas interpretariam `\r`, `\P`… como escape).

- O `relay.yaml` guarda **token e senhas** — restrinja a permissão do arquivo à
conta do serviço.

- Detalhes de captura de mudanças (CDC), allowlists, `statement_timeout_seconds`
e conta do serviço: `shift-connector/docs/operacao-relay.md` no repositório.
