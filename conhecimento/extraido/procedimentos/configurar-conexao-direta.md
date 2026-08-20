# Procedimento: criar uma conexão direta

**Fonte:** `m2p2:1-81` `[UI-OBSERVADA]` / `[VÍDEO]`, com conceito em `m2p1:10-25`.
**Escopo deste projeto: Oracle.** Os passos SQL Server de `m2p2:39-62` ficam registrados só como
contraste de campos, sem detalhar.

---

## Antes: quando a conexão direta serve

`[VÍDEO]` `m2p1:12-16` — o Shift roda **dentro da rede da Viasoft**. Daí decorrem três cenários:

| Onde está o banco | Conexão indicada |
|---|---|
| Dentro da rede Viasoft | **Direta** (IP + porta) |
| Na cloud Viasoft | **Direta**, via **TNS** — *"nosso padrão de comunicação"*, depende de liberações |
| Rede interna do cliente (on-premise) | **De borda** (relay). O Shift *"não vai ter como olhar diretamente"* a não ser que o cliente abra porta, *"o que é muito difícil"* |
| Banco em arquivo (`.DBF`, TXT, Excel) | **A partir de arquivos** |

`[VÍDEO]` Regra prática declarada: *"se você tem o banco dentro da tua máquina e a tua máquina
está dentro da rede da Viasoft, você não precisa de nenhuma conexão de borda"*. — `m2p2:42`

`[CONFIRMADO-MCP]` Confirmado na prática neste ambiente: `test_connection` retornou **SUCESSO**
nas duas conexões Oracle diretas (`192.168.90.218`), sem relay. Ver `divergencias.md §0`.

---

## Pré-requisito

`[VÍDEO]` O **sistema** precisa existir na lista de sistemas do espaço — inclusive quando é
concorrente. *"É importante, obviamente, aqui dentro do espaço, ter o concorrente, né, o
sistema."* — `m2p2:48`. Ver `procedimentos/criar-espaco.md` passo 5.

---

## Passos

1. **Escolha o nível onde a conexão vai morar** antes de criar: espaço, grupo econômico ou
   projeto. Isso define o dono e não é trivial de mudar depois.
   → O breadcrumb mostra em que nível você está. `m2p2:50`
   *Recomendação da doc:* `[CONFIRMADO-DOC]` na dúvida, **grupo econômico** — *"porque o banco
   quase sempre é do cliente"* (`conceitos.md § Conexão`).

2. **Vá em Conexões → Nova Conexão.**
   `[UI-OBSERVADA]` O menu de Conexões aparece com **ícone de tomada** nos três níveis. `m2p2:53`

3. **Nome.** Livre. `[VÍDEO]` Em aula: "ConstruShow", "Conexão ETrade".
   > Convenção sua, do plano §3: `<ambiente>-<sistema>-<base>`, ex.: `prd-oracle-erp`.

4. **Sistema.** Escolha da lista do espaço.
   → `[UI-OBSERVADA]` Ao escolher o sistema, **o tipo de banco vem preenchido** — *"como eu tipei
   o banco aqui, ele já traz aqui"*. `m2p2:16`

5. **Pública ou privada.**
   → `[UI-OBSERVADA]` **Privada é nível de usuário** — *"visível apenas para você"*. `m2p2:16`
   Pública desce para os níveis de baixo. Ver o modelo completo em `glossario.md § Conexão`.

6. **Ambiente:** Sandbox, Homologação ou Produção.
   → `[UI-OBSERVADA]` A interface mostra uma **temperatura / pontuação de risco de execução**:
   Produção acende como **"Hot"**. A aula descreve a gradação: *"Homologação, você pode fazer
   várias coisas. Sandbox, você pode até dropar esse banco"*. `m2p2:16`

   > ⚠️ **Não é cosmético.** `[CONFIRMADO-DOC]` `controle-de-acesso.md § Gate de produção`:
   > conexão com `environment = produção` faz a **execução do workflow exigir `ADMIN` do
   > workspace** — qualquer outro papel recebe `403`. A checagem olha as conexões referenciadas
   > **no momento da execução**. Escolher "Produção" aqui muda quem consegue rodar o fluxo.

7. **Usuário e senha** do banco.
   → `[CONFIRMADO-DOC]` A senha **nunca é devolvida** em leitura. `get_connection` verificado:
   só host, porta, banco e usuário.

8. **Método de conexão — Oracle tem dois:**

   **a) Easy Connect** — host, porta, banco.
   → `[VÍDEO]` Em aula: IP `192.168.91.89`, porta `1521`, banco `VIASOFT3`. `m2p2:19-25`

   > ⚠️ `[VÍDEO]` **Nunca use `localhost`.** *"Quando você estiver no Shift, aqui eu tô no
   > localhost, mas no Shift oficial você vai estar na internet. Então ele não vai entender
   > localhost. Então é sempre o teu IP, por mais que você esteja na rede da Viasoft."*
   > `m2p2:58`. Para descobrir: `ipconfig`.

   **b) TNS Description** — cole o **TNS Descriptor**.
   → `[VÍDEO]` É o caminho da cloud. O descriptor é *"o mesmo já utilizado para a conexão lá
   dentro do SQL Developer"*, onde aparece sob o **alias de rede**. `m2p2:17-18`
   Ver `m2p3` (lote 3) para o passo a passo de extrair do `tnsnames.ora`.

9. **Esquemas.** Selecione **quais schemas você quer ler**.
   → `[VÍDEO]` Um banco Oracle tem muitos; a lista de usuários no DBeaver mostra todos os
   possíveis. Em aula foram usados `VIASOFTMCP`, `VIASOFTBASE`, `VIASOFTFIN` — descritos como
   *"os principais pro funcionamento do ConstruShow"*. Também citados: `VIASOFTCTB` (contábil),
   `VIASOFTFISCAL`, `VIASOFTCP` (patrimônio). `m2p2:27-33`
   `[UI-OBSERVADA]` Deixar em branco faz pegar todos. `m2p2:60`

10. **Deixe desmarcado "Conexão via relay (banco on-premise)"** — só para conexão de borda.
    `m2p2:34`

11. **Clique em Criar.**
    → Mensagem *"Criada com sucesso"*.

12. **Teste.** Botão de **testar conexão (ícone de raio)**, disponível dentro do modal **e** na
    listagem.
    → Esperado: *"Conexão bem sucedida"*. `m2p2:36-38`

    `[CONFIRMADO-DOC]` **`VIEWER` já pode testar conexão** — está na matriz de capacidades.
    `[CONFIRMADO-MCP]` Pelo MCP: `test_connection` (SUCESSO/FALHA) e `diagnosticar_conexao`, que
    diz **em que etapa** falhou (DNS, TCP, handshake, login) e considera se passa por relay.

---

## Armadilha de usabilidade observada

`[VÍDEO]` O formulário **perde o conteúdo** ao sair sem salvar. O próprio instrutor tropeçou nisso
e comentou: *"não deixar sair sem salvar. Pô, vamos já implementar essa melhoria."* — `m2p2:31-33`

---

## Observações para o piloto

1. **O `Ambiente` da conexão decide quem executa.** Se a conexão Oracle do ERP for marcada como
   Produção, o fluxo agendado precisa rodar com `ADMIN` do workspace. **`[LACUNA]` Quem é o
   usuário efetivo numa execução disparada por `cron`?** Ver L29.
2. **`get_connection` não devolve o `environment`** — não há como conferir pelo MCP se as duas
   conexões Oracle deste ambiente são produção. E a doc lista o backfill desse campo como
   pendência aberta.
3. **Escolher os schemas corretos importa antes de começar:** a extração do piloto vai precisar
   dos schemas que contêm pedido, item, preço e margem. Ver L4.
