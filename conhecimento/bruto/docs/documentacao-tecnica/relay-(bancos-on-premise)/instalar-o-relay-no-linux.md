---
title: Instalar o Relay no Linux
---

Manual de instalação do `shift-relay` em servidores Linux. O relay é o
componente que roda **na infra do cliente** e permite que o Shift (nuvem)
alcance bancos de dados on-premise **sem abrir nenhuma porta de entrada no
firewall**.

O binário é o mesmo produto do Windows — Go estático, sem JVM, sem Oracle
Instant Client, sem dependência de distribuição. O que **não** existe no Linux é
a camada de embalagem: não há `Setup.exe`, wizard nem bundle `.zip`. Aqui o
instalador é o próprio binário: `shift-relay install` escreve a unit do systemd.

Para Windows, ver [Instalar o Relay no Windows](https://shift.viasoftcloud.com.br/docs/technical/relay/instalacao-windows).

---

## 1. O que o relay faz

```
  Rede do cliente                              Nuvem
  ┌──────────────┐     ┌─────────────┐        ┌───────────────┐
  │ banco(s)     │◄────│ shift-relay │ ──wss─►│ shift-gateway │◄── Shift
  │ ERP/Oracle/… │ LAN │  (systemd)  │ saída  │               │
  └──────────────┘     └─────────────┘  443   └───────────────┘
```

- **Só tráfego de saída.** O relay sempre inicia a conexão; o gateway nunca
alcança a rede do cliente por conta própria.

- **Burro de propósito.** Não conhece tipo de banco, não tem driver instalado,
não vê usuário/senha/SQL do fluxo — para ele é tudo TCP.

- **Allowlist local.** O relay só alcança os `host:porta` declarados no config.

---

## 2. Antes de começar

### No servidor

| Item | Requisito |
| --- | --- |
| Sistema | Qualquer distribuição com **systemd** (RHEL/Oracle Linux 8+, Debian 11+, Ubuntu 20.04+, SUSE…) |
| Arquitetura | `x86_64` (há build para `arm64` — ver seção 3) |
| Permissão | `root` (ou `sudo`) para registrar o serviço |
| Disco | ~100 MB para o binário + espaço no `data_dir` (logs e, se usar a borda, Parquets — podem ser grandes) |
| Rede | Saída HTTPS/443 liberada para o host do gateway (`tunnel-shift.viasoftcloud.com.br`) |
| Relógio | Sincronizado (`chronyd`/`systemd-timesyncd`) |

> **Proxy corporativo:** o relay **não** usa `HTTP_PROXY`/`HTTPS_PROXY` para a
> conexão `wss`. Em rede com proxy explícito obrigatório, libere saída direta na
> 443 para o host do gateway.

Não é preciso liberar nada no `firewalld`: regras de entrada não se aplicam
(o relay não escuta porta) e a política padrão de saída é permissiva.

### Informações que a Viasoft/o operador do Shift precisa fornecer

- **Token do relay** (gerado no Shift, exibido **uma única vez**).

- **Nome do relay**.

- Para **cada banco** exposto: um **número de destino** (`remote_port`), que vale
só dentro desta instalação — comece em `15432` e siga (`15433`, …). Não há nada
a combinar com o Shift: ele resolve o endereço do lado dele sozinho.

- Se o destino também vai **aplicar/extrair na borda** (Oracle): service name,
usuário, senha e a lista de tabelas permitidas.

### Passo 0 — Criar o relay no Shift

No Shift, com perfil **ADMIN**: **Configurações → Relays → Novo relay** → dê um
nome → **Criar** → **copie o token** (mostrado uma vez só).

---

## 3. Passo 1 — Obter o binário

O `Makefile` do `shift-connector` já cross-compila para Linux:

```
cd shift-connector && make relay
```

Saída: `bin/shift-relay-linux-amd64` (e o `.exe` do Windows). Para ARM:

```
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -ldflags "-s -w -X main.version=$(git describe --tags --always)" -o bin/shift-relay-linux-arm64 ./cmd/shift-relay
```

Sem Go na máquina de build, o `Dockerfile` do `shift-connector` compila os dois
binários numa imagem `distroless` — dá para extrair o `/bin/shift-relay` dela.

Transfira o binário para o servidor (scp/pendrive) e confira o hash com o que
foi gerado (`sha256sum`).

---

## 4. Passo 2 — Instalar os arquivos

Como `root`:

```
install -m 0755 shift-relay-linux-amd64 /usr/local/bin/shift-relay
mkdir -p /etc/shift-relay /var/lib/shift-relay
```

- `/usr/local/bin/shift-relay` — o binário.

- `/etc/shift-relay/relay.yaml` — a configuração.

- `/var/lib/shift-relay` — o `data_dir`: logs, estado dos jobs, Parquets da
borda e o watermark das capturas. **Fica fora do caminho do binário de
propósito**, para sobreviver a atualizações (apagar os offsets faria
reprocessar tudo).

Em SELinux *enforcing*, `/usr/local/bin` já tem o contexto correto para
execução por serviço. Se optar por outro caminho, rode `restorecon -v` no
binário depois de copiar.

---

## 5. Passo 3 — Escrever o `relay.yaml`

```
cat > /etc/shift-relay/relay.yaml <<'YAML'
gateway_url: wss://tunnel-shift.viasoftcloud.com.br/tunnel
relay_name: erp-matriz          # = nome do relay no Shift
token: COLE-O-TOKEN-AQUI
fingerprint: ""                 # sha256 do cert do gateway (pin opcional)
tls_max_version: ""             # vazio = 1.3 com fallback automático p/ 1.2
log_level: info                 # info | debug (debug é volumoso)
keepalive_seconds: 25
data_dir: /var/lib/shift-relay

destinations:
  - name: oracle-erp            # alias usado pelos jobs da borda
    remote_port: 15432          # número do destino nesta instalação
    host: 10.0.0.30             # como ESTE servidor vê o banco
    port: 1521
YAML

chmod 600 /etc/shift-relay/relay.yaml
```

O arquivo guarda **token** (e senhas, se usar a borda) — o `chmod 600` não é
opcional.

Destino que também aplica/extrai na borda (Oracle) ganha as credenciais e a
allowlist:

```
  - name: oracle-erp
    remote_port: 15432
    host: 10.0.0.30
    port: 1521
    apply_dialect: oracle
    apply_service: ORCLPDB1
    apply_user: SHIFT_LOAD
    apply_password: "<senha>"
    apply_allowlist:
      - "VIASOFTBASE.*"
    extract_dialect: oracle
    extract_service: ORCLPDB1
    extract_user: SHIFT_READ
    extract_password: "<senha>"
    extract_allowlist:
      - "CONSTRUSHOW.*"
```

> Allowlist **vazia ou ausente = nada é permitido** (secure by default): o
> destino com credencial mas sem allowlist recusa toda carga/leitura.

---

## 6. Passo 4 — Validar a configuração

```
shift-relay check -config /etc/shift-relay/relay.yaml
```

O `check` valida o YAML (campos obrigatórios, portas, duplicidade de
`remote_port`) e, havendo capturas configuradas, prova que o `data_dir` é
gravável e lista os pré-requisitos de CDC de cada origem. Sai com código ≠ 0 se
alguma verificação falhar.

---

## 7. Passo 5 — Registrar o serviço no systemd

```
shift-relay install -config /etc/shift-relay/relay.yaml
shift-relay start
```

O `install` escreve `/etc/systemd/system/ViasoftShiftServiceRelay.service`, roda
`systemctl enable` (sobe no boot) e `daemon-reload`. A unit gerada usa
`Restart=always` com `RestartSec=120` — depois de uma queda, o systemd sobe o
processo de novo em 2 minutos (no Windows o SCM usa 10s; aqui é o padrão do
template).

> O `install` **falha com `Init already exists`** se a unit já existir. Para
> reinstalar (ex.: mudou o caminho do config), rode `shift-relay uninstall`
> antes.

Conferir:

```
systemctl status ViasoftShiftServiceRelay
journalctl -u ViasoftShiftServiceRelay -n 50
tail -n 40 /var/lib/shift-relay/logs/shift-relay.log
```

O **ciclo de vida** (start/stop/panic) vai para o journald; o **log detalhado do
túnel** vai para `<data_dir>/logs/shift-relay.log`, rotacionado (20 MB por
arquivo, 5 backups, 30 dias). Do lado do Shift, a tela **Relays** passa a
mostrar o relay ativo, com a versão anunciada no handshake.

### Variante: unit própria com usuário dedicado

O `install` gera uma unit que roda como **root** (o binário não define usuário
de serviço). Se a política do cliente exigir usuário dedicado, **não** use o
`shift-relay install` — escreva a unit à mão:

```
useradd --system --no-create-home --shell /usr/sbin/nologin shift-relay
chown -R shift-relay:shift-relay /var/lib/shift-relay
chown shift-relay:shift-relay /etc/shift-relay/relay.yaml

cat > /etc/systemd/system/shift-relay.service <<'UNIT'
[Unit]
Description=Shift Relay
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/shift-relay -config /etc/shift-relay/relay.yaml
User=shift-relay
Restart=always
RestartSec=10
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/shift-relay

[Install]
WantedBy=multi-user.target
UNIT

systemctl daemon-reload
systemctl enable --now shift-relay
```

Rodando sob systemd o relay continua gravando o log detalhado no arquivo (ele
detecta que não há terminal); o journald recebe o ciclo de vida.

### Variante: container

Se o cliente já roda Docker/Kubernetes, a imagem do `Dockerfile` do
`shift-connector` serve direto — sem systemd:

```
docker run -d --name shift-relay --restart unless-stopped \
  --network host \
  -v /etc/shift-relay/relay.yaml:/etc/shift-relay/relay.yaml:ro \
  -v /var/lib/shift-relay:/var/lib/shift-relay \
  <imagem> /bin/shift-relay -config /etc/shift-relay/relay.yaml
```

- Em container o relay se detecta como interativo e manda os logs para
**stdout** (`docker logs`), além do arquivo no `data_dir`.

- **Rede:** o container precisa alcançar os bancos da LAN. `--network host` é o
caminho curto quando os destinos são `localhost`/rede do host; com rede bridge,
garanta rota até os `host:porta` dos destinos.

---

## 8. Passo 6 — Apontar a conexão no Shift para o relay

No Shift, ao cadastrar/editar a conexão do banco:

1. Escolha o **relay** na conexão.

2. Informe a **Porta do destino no relay** — exatamente o mesmo número do
`remote_port`. O endereço do lado do Shift é gerado por ele.

3. **Testar conexão.**

---

## 9. Operação do dia a dia

```
systemctl status ViasoftShiftServiceRelay      # estado
systemctl restart ViasoftShiftServiceRelay     # reiniciar
journalctl -u ViasoftShiftServiceRelay -f      # ciclo de vida ao vivo
tail -f /var/lib/shift-relay/logs/shift-relay.log   # túnel/jobs

shift-relay version                            # versão do binário
shift-relay check -config /etc/shift-relay/relay.yaml
```

Os subcomandos `start`/`stop`/`status`/`uninstall` do binário são atalhos para o
`systemctl` equivalente — use o que for mais familiar ao TI do cliente.

### Bundle de diagnóstico para o suporte

```
shift-relay diag -config /etc/shift-relay/relay.yaml
```

Gera `shift-relay-diag-<timestamp>.zip` no diretório atual com logs, o
`relay.yaml` **sanitizado** (token/senhas viram `***`), snapshot de versão/
capabilities/destinos e um teste de conectividade (DNS, TCP no gateway, skew de
relógio). Funciona com o serviço **rodando ou parado** e não contém credencial
nem linha de dado.

---

## 10. Atualizar e desinstalar

**Atualizar** (não há auto-update):

```
shift-relay stop
install -m 0755 shift-relay-linux-amd64 /usr/local/bin/shift-relay
shift-relay start
```

Config e `data_dir` ficam intactos — o relay retoma os offsets e **não
reprocessa** o que já foi lido. Só é preciso `uninstall`/`install` se o caminho
do binário ou do config mudar (a unit guarda esses caminhos).

**Desinstalar:**

```
shift-relay stop
shift-relay uninstall            # remove a unit e desabilita no boot
rm -f /usr/local/bin/shift-relay /etc/shift-relay/relay.yaml
# /var/lib/shift-relay é preservado — apague só se for descartar logs e offsets
```

Com a unit própria da seção 7: `systemctl disable --now shift-relay` e remova o
arquivo da unit.

---

## 11. Problemas comuns

| Sintoma | Causa provável / o que fazer |
| --- | --- |
| `install` responde `Init already exists` | A unit já existe. Rode `shift-relay uninstall` antes de reinstalar |
| Serviço não sobe; journal mostra `config inválida` | Campo obrigatório faltando (`gateway_url`, `relay_name`, `token`) ou destino sem `host`/portas. Rode `shift-relay check` |
| `systemctl` não acha a unit | O nome é `ViasoftShiftServiceRelay.service` (ou o da unit própria, se você a escreveu) |
| Log repete erro conectando no gateway | Saída 443 bloqueada, DNS não resolve, ou proxy explícito obrigatório (não suportado). Confirme com o `diag` |
| Conecta e cai em seguida | Token revogado/errado, ou `relay_name` diferente do cadastrado no Shift |
| Demora ~2 min para voltar após um crash | `RestartSec=120` da unit gerada. Use a unit própria com `RestartSec=10` se precisar de retomada mais rápida |
| `remote_port já usada` | Dois destinos com o mesmo número **nesta** instalação — dentro do relay o número é a chave do destino, então cada banco precisa do seu |
| Conexão do Shift falha com o relay online | Número do destino no cadastro ≠ `remote_port`; ou o banco recusa conexão vinda do IP do servidor do relay |
| Job da borda falha com "tabela não permitida" | Allowlist vazia ou que não cobre a tabela — sem allowlist o relay **nega tudo**, por design |
| Permissão negada ao gravar offsets/logs | O `data_dir` não é gravável pelo usuário do serviço. Ajuste o `chown` (relevante na variante com usuário dedicado) |

---

## 12. Referência

- Captura de mudanças (CDC), allowlists, `statement_timeout_seconds`, conta de
serviço e estratégia de atualização: `shift-connector/docs/operacao-relay.md`.

- Pré-requisitos de CDC por banco: `shift-connector/docs/cdc-postgres.md` e
`shift-connector/docs/cdc-sqlserver.md`.

- Exemplo completo de configuração: `shift-connector/configs/relay.example.yaml`.
