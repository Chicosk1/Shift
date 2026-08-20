# Nó `workflow_input` — Entrada do Fluxo (Entrada do Sub-workflow)

| | |
|---|---|
| **Tipo (MCP)** | `workflow_input` |
| **Rótulo na interface** | Entrada do Fluxo / Entrada do subfluxo |
| **Categoria** | `subflow` `[CONFIRMADO-MCP]` — mas ver divergência abaixo |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — (nenhum sucessor declarado nas fontes) |

> **É o nó que declara "este fluxo é um subfluxo".** Não é só um ponto de entrada: a presença dele
> muda o que o fluxo é e como pode ser executado.

## ⚠️ DIVERGÊNCIA — categoria

**Ambas registradas, nenhuma escolhida:**

- `[CONFIRMADO-MCP]` `describe_node('workflow_input')` → **Categoria: `subflow`**. E
  `list_nodes(category='trigger')` retorna **apenas** `manual`, `webhook` e `cron` — o
  `workflow_input` **não está** entre os triggers.
- `[CONFIRMADO-DOC]` `nos/entrada-do-fluxo.md` → **"Categoria: Gatilho"**.
- `[VÍDEO]` A aula o trata como gatilho: *"Vamos ver o próximo nó aqui de gatilho"* na sequência
  em que ele aparece, e *"quando ele é um subfluxo, ele sempre terá o nó de **gatilho** Entrada de
  subfluxo"*. — `m3-2A:49`, `m3-2A:61`

**Leitura provável (`[INFERIDO]`):** ele é gatilho na **função** (é o nó que inicia o fluxo, e a
regra de "um gatilho por fluxo" se aplica a ele) e `subflow` na **taxonomia do catálogo**, onde
mora junto de `workflow_output` e `call_workflow`. Consequência prática de saber disso: ao procurar
o nó por categoria no MCP, buscar em `subflow`, não em `trigger`.

## O que faz

`[CONFIRMADO-MCP]` Ponto de entrada interno de um sub-workflow. **Expõe os parâmetros declarados
no `input_schema`** para os nós internos consumirem. — `describe_node('workflow_input')`

`[CONFIRMADO-DOC]` Define o ponto de entrada de um **sub-fluxo** — um workflow reutilizável
invocado por outros via nó **Chamar Fluxo**. Quando o sub-fluxo é chamado, os parâmetros
fornecidos pelo chamador chegam aqui. — `nos/entrada-do-fluxo.md § Descrição`

`[VÍDEO]` A aula é a fonte mais clara sobre o **propósito**, e vale citar:

> *"Ela sinaliza que esse fluxo, ele é um subfluxo, ou seja, ele sempre e só deve ser acionado
> através de outro fluxo."* — `m3-2A:59`

E o motivo de existir, com o exemplo de migração do ConstruShow: antes de cadastrar um produto é
preciso ter subgrupo, grupo, seção, departamento, marcas e unidades. *"Se eu coloco tudo isso
dentro de um fluxo, isso me impossibilita a reutilização. Se eu for fazer outro processo cujo
cadastro de grupo é importante, eu teria que refazer ele lá dentro ou dar um control C, control V
(...) Quando ele é um subfluxo, eu torno ele reutilizável. Ou seja, quando eu defino ele como um
subfluxo, ele pode ser chamado por qualquer fluxo, desde que passe os parâmetros que ele espera."*
— `m3-2A:56-58`

`[UI-OBSERVADA]` **O ícone do fluxo muda na listagem** quando ele tem este nó: *"Se vocês notarem
aqui, ele mudou o ícone. Ele já identificou que é um subfluxo."* — `m3-2A:61`

## Quando usar

`[CONFIRMADO-MCP]`
- Receber parâmetros de entrada de um nó `call_workflow` pai.
- Declarar o contrato de entradas que o sub-workflow aceita.

`[CONFIRMADO-DOC]` *"Use este nó sempre que quiser criar um workflow que funcione como uma
**sub-rotina**: uma lógica encapsulada com entradas e saídas bem definidas, reutilizável em
múltiplos outros workflows."* — `nos/entrada-do-fluxo.md § Descrição`

`[VÍDEO]` Critério de decisão que o instrutor usa, por analogia com orientação a objetos: o
cadastro de grupo *"é um objeto imutável, então ele sempre vai ser assim"* — lógica que não varia
entre casos é candidata a subfluxo. O que varia é a **origem** do dado: *"a parte de inserção e
tratamento é a mesma (...) pode vir de um CSV, de uma URL, de uma planilha do Google Sheets"*.
— `m3-2A:57`, `m3-2A:69-70`

## Parâmetros de configuração

`[CONFIRMADO-MCP]` + `[CONFIRMADO-DOC]` — as duas fontes concordam: **um único campo.**

| Parâmetro | Tipo | Padrão | Descrição |
|---|---|---|---|
| `output_field` | string | `data` | Nome do campo da saída onde os parâmetros de entrada são expostos |

— `describe_node('workflow_input')`, `nos/entrada-do-fluxo.md § Configurações`

### Onde mora o `input_schema` (o "esquema de IO")

**O contrato de entrada NÃO é configuração deste nó.** `[CONFIRMADO-MCP]` O contrato do nó cita o
`input_schema` (*"expõe os parâmetros declarados no `input_schema`"*) mas **não o lista** entre os
campos configuráveis.

`[UI-OBSERVADA]` A aula mostra onde ele fica: num painel do **fluxo**, não do nó — *"aqui dentro
de Schema de... e Input/Output ou IO aqui, input/output, ou seja, entrada, o que esse cara espera,
e saída, o que aquele outro cara vai fazer"*. — `m3-2A:74`

`[INFERIDO]` Corroborado pela superfície de ferramentas do MCP, que expõe
`pending_set_io_schema` como operação **de fluxo** (não de nó) — consistente com o esquema ser
propriedade do workflow. **Ferramenta de escrita, não executada.**

`[UI-OBSERVADA]` Como declarar, passo a passo observado em aula (`m3-2A:75-79`):

1. Escolher o **tipo** do que se espera. Para um campo único usam-se os tipos escalares; *"agora
   se ele vai me retornar uma tabela com múltiplos campos, eu tenho que definir como um **table
   reference**"*.
2. Adicionar as **colunas**, cada uma com **tipo** e a marca de **obrigatório**. Tipos vistos na
   tela: **inteiro**, **string**, **booleano**, **número**, e um campo de data (`data_admissao`).
3. Dar um **nome à tabela** (na aula, `Funcionarios`) e **aplicar**.

`[UI-OBSERVADA]` **O schema gera um mock de teste automaticamente**, e ele é baixável:
*"Como eu tenho o schema de input, ele já traz aqui um mock para teste do que ele espera (...) eu
posso até vir aqui, pegar esses dados e baixar eles. Ele baixou o preview. E agora eu posso usar
aqui pra testar. Como se tivesse chegado aqui já os nossos dados."* — `m3-2A:80-81`

> Esse é **o** truque de teste de subfluxo: declarar o schema, baixar o preview gerado, e
> recarregá-lo como mock no próprio nó de entrada. Sem isso não há como testar o subfluxo
> isolado, porque ele não aceita disparo manual.

## Entradas esperadas

**Nenhuma no canvas.** Os dados entram **de fora do fluxo**, via `call_workflow` do fluxo pai.

`[CONFIRMADO-DOC]` **Não pode ser disparado manualmente pelo botão Play** como os outros
gatilhos. *"Ele só é acionado quando outro workflow invoca este fluxo via nó Chamar Fluxo."*
— `nos/entrada-do-fluxo.md § Descrição`

> ⚠️ **DIVERGÊNCIA — dá ou não para executar no editor?** **Ambas registradas:**
>
> - `[CONFIRMADO-DOC]` *"Este nó **não pode** ser disparado manualmente pelo botão Play."*
> - `[VÍDEO]` A aula executa, sim, em **modo teste**, usando o mock: *"Se você executar aqui ele
>   vai executar, em teste. Mas se você tiver um mock de teste. **Em produção ele só executa
>   chamando a partir de outro** subfluxo."* — `m3-2A:110`. E antes: *"Agora se eu tentar
>   executar, não vai dar nada, não tem nada, tipo tá vazio, porque não tenho o mock de teste."*
>   — `m3-2A:88`
>
> **Leitura provável (`[INFERIDO]`):** a doc descreve o comportamento **em produção**; a aula
> mostra que **em teste, com mock carregado, a execução no editor funciona**. Não é contradição de
> fato, é omissão da doc sobre o modo teste — mas como a diferença decide se dá para testar um
> subfluxo isolado, fica sinalizada.

`[CONFIRMADO-MCP]` **Em produção o mock é ignorado**, e isso a aula afirma com ênfase:
*"em produção ele **nunca** considera o mock de teste, por mais que eu tenha o mock de teste, ele
não vai executar. Então, é só em teste."* — `m3-2A:89` `[VÍDEO]`

## Saídas produzidas

`[CONFIRMADO-DOC]` — `nos/entrada-do-fluxo.md § Saída produzida`

```json
{
  "status": "completed",
  "output_field": "data",
  "data": { "...parâmetros enviados pelo chamador..." }
}
```

Os nós seguintes referenciam via `upstream_results.<nodeId>.data` — ou pelo nome que estiver em
`output_field`. Exemplo da doc com `output_field: entrada`:

```
{{ upstream_results.<nodeId>.entrada.pedido_id }}
```

## Isolamento de contexto — a regra que mais surpreende

`[CONFIRMADO-DOC]` O sub-fluxo roda em contexto **completamente isolado** do workflow pai
(`nos/entrada-do-fluxo.md § Isolamento de contexto`):

- **Não herda** variáveis, conexões ou resultados de nós do pai.
- **Exceção — auto-forward:** variáveis com o **mesmo nome** declaradas tanto no pai quanto no
  sub-fluxo são **propagadas automaticamente**. Isso evita remapeamento manual de conexões de
  banco compartilhadas.
- O chamador pode **sobrescrever** os valores propagados via `variable_values` no nó Chamar Fluxo.

`[CONFIRMADO-MCP]` O contrato de `call_workflow` diz a mesma coisa do outro lado: *"o sub-workflow
roda em isolamento (contexto próprio, sem herdar upstream do pai)"*.

## Erros comuns

`[CONFIRMADO-DOC]` — `nos/entrada-do-fluxo.md § Limites e guardrails`

- Um workflow com Entrada do Fluxo pode ter **apenas um** nó deste tipo.
- **Parâmetros obrigatórios não fornecidos** pelo chamador → **erro em tempo de execução no nó
  Chamar Fluxo** (ou seja: a falha aparece no **pai**, não no subfluxo).
- **Ciclos de chamada** (A invoca B que invoca A) são detectados e bloqueados pelo runtime, via
  `call_stack` + `max_depth`.
- **Rehidratação de datasets upstream:** se o chamador mapear linhas de um nó materializado em
  DuckDB (ex.: resultado de um `filter`), o runtime carrega **até 1.000 linhas** antes de resolver
  o mapeamento. **Para volumes maiores, use o nó Loop (For Each) em vez de sub-fluxo.**

> ⚠️ **O teto de 1.000 linhas é o limite mais perigoso deste nó** para o piloto — é silencioso por
> natureza (não é erro, é truncamento na rehidratação) e a doc oferece uma saída explícita
> (`loop`). Ver observações abaixo.

`[VÍDEO]` Erro observado por omissão, do outro lado do par: se o subfluxo **não define saída**, o
pai não recebe nada. *"Por que que ele não trouxe nada aqui? Porque lá dentro do nosso subfluxo a
gente não definiu uma saída. Então quando não tem uma saída ele só executa."* — `m3-2A:98`

`[CONFIRMADO-DOC]` `call_workflow` **não é exportável** para SQL/Python (HTTP 422). Como todo
subfluxo existe para ser chamado, isso afeta o par inteiro na prática.
— `guias-de-uso/exportar-e-importar.md § Cobertura V1`

## Observabilidade

`[CONFIRMADO-DOC]` A execução do sub-fluxo gera um **`execution_id` próprio, com prefixo `sub-`**,
visível nos logs. Cada invocação é rastreável independentemente do workflow pai.
— `nos/entrada-do-fluxo.md § Observabilidade`

`[UI-OBSERVADA]` Como isso aparece na tela, em aula: na aba **Execuções**, o subfluxo tem
tratamento próprio — *"quando é um subfluxo, aqui ele é um pouco diferente. Ele vem aqui, ele te
mostra: esse aqui é um subfluxo, foi completado, tem 3 nós lá dentro, e você pode vir aqui e ver
as **subexecuções**"*. — `m3-2A:97`

`[UI-OBSERVADA]` E a diferença entre modos: em **teste** o canvas mostra tudo o que aconteceu
dentro do subfluxo; em **produção** o detalhamento só existe na aba de execuções. Motivo declarado
pelo instrutor: performance — *"é muito caro eu ficar vindo aqui e atualizando a tela só pra você
acompanhar. A ideia disso aqui é que você rode e vai tomar um café."* — `m3-2A:96`

## Exemplo

`[CONFIRMADO-DOC]` Sub-fluxo que recebe um `pedido_id` e retorna o status
(`nos/entrada-do-fluxo.md § Exemplo de uso`):

```json
// No nó Entrada do Fluxo
{"output_field": "entrada"}
```

```
// No nó seguinte (ex.: SQL)
{{ upstream_results.<nodeId>.entrada.pedido_id }}
```

`[VÍDEO]` O exemplo completo da aula é o subfluxo **`Funcionarios Core`**: extraiu-se do fluxo
`Inserir Funcionários` a parte de tratamento + inserção, deixando a origem do dado fora. Schema de
input declarado como **table reference** chamada `Funcionarios`, com colunas `ID` (inteiro,
obrigatório), `nome` (string), `ativo` (booleano), `email` (string), `cidade` (string), `salario`
(número), `departamento` (string), `data_admissao`. Dentro: um Mapeamento (maiúsculo em nome,
cidade e departamento; remover acentos) e uma inserção com **UPSERT** — escolhido explicitamente
*"para ele não dar erros"* com dados já existentes. — `m3-2A:72-86`

## Observações para o piloto de margem

1. **O teto de 1.000 linhas na rehidratação é a decisão de arquitetura mais importante aqui.**
   Se o piloto quebrar o fluxo de margem em subfluxos (ex.: `sub-calcular-preco`,
   `sub-gravar-preco`) e passar o **conjunto de pedidos** como parâmetro, o mapeamento de entrada
   trunca em 1.000 linhas. A doc manda usar `loop` (For Each) para volumes maiores — o que troca
   uma chamada por N chamadas e muda completamente o perfil de desempenho de uma rotina que roda a
   cada 5 minutos. `[LACUNA]` Falta descobrir se o limite dispara erro ou trunca em silêncio; a
   doc diz *"carrega até 1.000 linhas"*, o que soa a truncamento.

2. **Recomendação derivada:** para o piloto, prefira subfluxo para **lógica sem volume**
   (parametrização, decisão, registro de auditoria de uma linha) e mantenha o **caminho dos dados
   em um único fluxo**, onde o dataset fica materializado em DuckDB sem rehidratação.

3. **O isolamento de contexto não herda conexões** — mas o **auto-forward por nome de variável**
   resolve: declare a variável de conexão com **o mesmo nome** no pai e no subfluxo e ela é
   propagada. Sem isso, cada subfluxo precisaria remapear a conexão Oracle.

4. **Não é o gatilho do piloto.** O piloto usa `cron`, e **um fluxo só pode ter um gatilho**
   (`m3-2A:5`). Logo: **o fluxo agendado não pode ser um subfluxo**, e um subfluxo não pode ser
   agendado. Se o piloto virar `cron` + `call_workflow`, o agendamento fica no pai e a lógica no
   filho.

5. **Testar subfluxo depende do mock**, e o mock **não vale em produção** (`m3-2A:89`). O ciclo de
   validação do piloto tem que prever isso: o subfluxo é testado com preview baixado do schema, e
   a validação em produção só acontece pelo pai.
