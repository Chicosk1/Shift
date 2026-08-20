# Nó `cron` — Agendamento (Cron)

| | |
|---|---|
| **Tipo (MCP)** | `cron` |
| **Rótulo na interface** | Agendamento / Agendamento (Cron) |
| **Categoria** | `trigger` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — (nenhum sucessor declarado nas fontes) |

> **Gatilho do piloto de margem.** É este nó que faz o fluxo de ajuste de preço rodar a cada
> 5 minutos. Tudo o que está marcado como `[LACUNA]` aqui é lacuna **do caminho crítico**.

## O que faz

`[CONFIRMADO-MCP]` Dispara o workflow automaticamente em horários recorrentes definidos por
expressão cron. Retorna timestamp do disparo como output. — `describe_node('cron')`

`[CONFIRMADO-DOC]` A programação é configurada por uma interface visual que **gera** a expressão
cron equivalente — não é necessário saber escrever cron manualmente.
— `nos/agendamento-(cron).md § Descrição`

`[VÍDEO]` Em aula: *"para situações de fluxos recorrentes, todo dia eu quero executar tal coisa,
a cada 10 minutos eu quero executar tal coisa"*. O instrutor confirma o mesmo mecanismo:
*"óbvio, ele trabalha com a expressão Cron (...) mas aí para facilitar aqui você não precisa saber
a linguagem Cron, você só vai definindo aqui"*. — `m3-2A:17-18`

## Pré-condição que invalida o nó

`[CONFIRMADO-DOC]` **O agendamento só fica ativo quando o workflow está em modo Produção E
Publicado.** Em modo de teste (draft), o nó pode ser executado manualmente mas **não dispara
sozinho**. — `nos/agendamento-(cron).md § Descrição`

`[VÍDEO]` A aula diz a mesma coisa, em sequência operacional: *"se você vir aqui salvar,
publicar, obviamente, deixar em produção, então isso aqui todos os dias, às 10 da manhã, o
sistema vai executar esses caras aí"*. — `m3-2A:23`

Procedimento completo em `procedimentos/publicar-e-agendar.md`.

## Quando usar

`[CONFIRMADO-MCP]`
- Executar o fluxo diariamente em horário fixo.
- Sincronizar dados em intervalos regulares (ex.: a cada hora).
- Agendar jobs noturnos ou de fim de semana.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('cron')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `cron_expression` | string | **sim** | Expressão cron (ex.: `0 8 * * 1-5` = dias úteis às 8h) |

**É o único parâmetro do contrato.** Não existe campo de fuso horário, de dias da semana, de
piso de intervalo ou de política de concorrência no contrato do nó.

### ⚠️ DIVERGÊNCIA — contrato tem 1 campo, a interface tem 6 (lacuna **L15**)

As duas fontes discordam sobre o que é configuração do nó. **Ambas registradas:**

- `[CONFIRMADO-MCP]` `describe_node('cron')` declara **um único parâmetro**, `cron_expression`
  (string, obrigatório). **Sem piso de intervalo.**
- `[CONFIRMADO-DOC]` `nos/agendamento-(cron).md § Configurações` descreve **um formulário
  inteiro**: Frequência (presets), Horário, Fuso horário, Dias da semana, Meses, Dias do mês.

**Leitura provável (`[INFERIDO]`):** não é conflito de fato, é diferença de camada. Os campos da
doc são **geradores** da expressão — a própria doc diz que a interface *"gera a expressão cron
equivalente"* e que o valor *"é gravado no campo `cron_expression`"*. O contrato guarda só o
resultado.

**Consequência que importa ao piloto:** o menor intervalo oferecido é **5 minutos**, e esse piso
é **da interface, não do contrato**. `cron_expression` aceita qualquer expressão cron válida.
`[LACUNA]` Falta descobrir se a API valida um intervalo mais curto que 5 minutos (ex.:
`* * * * *`) ou se aceita e o agendador executa — e, se aceitar, se há limite de execuções por
janela. Como o piloto roda exatamente no piso de 5 minutos, a pergunta é operacional, não
acadêmica.

### Presets de frequência (camada de interface)

`[CONFIRMADO-DOC]` — `nos/agendamento-(cron).md § Frequência`

| Opção | Expressão gerada |
|---|---|
| A cada 5 minutos | `*/5 * * * *` |
| A cada 10 minutos | `*/10 * * * *` |
| A cada 15 minutos | `*/15 * * * *` |
| A cada 30 minutos | `*/30 * * * *` |
| A cada hora | `0 * * * *` |
| A cada 2 horas | `0 */2 * * *` |
| A cada 3 horas | `0 */3 * * *` |
| A cada 6 horas | `0 */6 * * *` |
| Horário específico | `<min> <hora> * * *` |

`[UI-OBSERVADA]` A aula confirma os presets pelo rótulo na tela: *"Se é a cada 10 minutos,
5 minutos, 6 horas, um horário específico"*. — `m3-2A:17`

### Demais campos da interface

`[CONFIRMADO-DOC]` — `nos/agendamento-(cron).md § Demais campos`

| Campo | Tipo | Padrão | Observação |
|---|---|---|---|
| **Horário** | `HH:MM` | `09:00` | Visível apenas com "Horário específico" |
| **Fuso horário** | string | `America/Sao_Paulo` | Referência para calcular a hora do disparo |
| **Dias da semana** | seleção múltipla | todos | Desmarque "Toda a semana" para escolher dias |
| **Meses** | seleção múltipla | todos | Desmarque "Todos os meses" para restringir |
| **Dias do mês** | seleção múltipla (1–31) | todos | Desmarque "Todos os dias" para escolher datas |

`[UI-OBSERVADA]` Os mesmos rótulos aparecem na tela da aula: *"Fuso horário, geralmente você vai
usar o America/Sao_Paulo"*; *"se é todo dia da semana, ou se é só, sei lá, de segunda... segunda a
quinta"*; *"Ou se é todos os meses"*. — `m3-2A:18`

`[UI-OBSERVADA]` **Prévia das próximas execuções** — aba que exibe as 5 próximas disparadas
previstas, convertidas para o horário local do navegador. Serve para conferir o agendamento
**antes de publicar**. — `nos/agendamento-(cron).md § Prévia das próximas execuções`
`[CONFIRMADO-DOC]`

## Entradas esperadas

**Nenhuma.** É gatilho — só tem saída. `[CONFIRMADO-DOC]` `conceitos.md § Nó` generaliza a regra
para todos os gatilhos (ver `nos/manual.md`).

`[VÍDEO]` A aula estabelece a regra estrutural do gatilho: *"os nós de gatilhos é aquilo que
define como aquele fluxo será iniciado. Ele é obrigatório ter (...) Eu não consigo ter dois, eu
não posso ter dois nós de gatilho dentro do mesmo fluxo e eu não posso não ter eles dentro do
fluxo."* — `m3-2A:5`

## Saídas produzidas

`[CONFIRMADO-DOC]` O nó **não carrega dados externos** — apenas sinaliza que o fluxo foi iniciado
pelo agendador. — `nos/agendamento-(cron).md § Saída produzida`

```json
{
  "trigger_type": "cron",
  "status": "triggered",
  "cron_expression": "0 9 * * 1-5",
  "triggered_at": "2025-06-10T09:00:00+00:00",
  "data": {}
}
```

`[CONFIRMADO-DOC]` **`triggered_at` está sempre em UTC** (ISO 8601). Converta-o em um nó
posterior se precisar de outro fuso. — mesma seção.

> **Armadilha de fuso.** O campo **Fuso horário** default é `America/Sao_Paulo` e governa *quando*
> o disparo acontece; `triggered_at` sai em **UTC** e é o que os nós seguintes leem. Um fluxo que
> use `triggered_at` para montar filtro de data compara UTC com dado gravado em horário local —
> 3 horas de defasagem. Para o piloto, que corta pedidos por janela de tempo, isso é erro
> silencioso de resultado, não erro de execução.

`[CONFIRMADO-MCP]` O contrato descreve a saída como *"timestamp do disparo"* — consistente com a
doc, mas sem detalhar os outros campos.

## Erros comuns

`[CONFIRMADO-DOC]` Erros de **validação**, que impedem salvar o agendamento
(`nos/agendamento-(cron).md § Limites e guardrails`):

- Selecionar "Dias da semana" sem marcar "Toda a semana" e não escolher nenhum dia → erro de
  validação, **o agendamento não é salvo**.
- O mesmo vale para **Meses** e **Dias do mês**.
- Hora fora de 0–23 ou minuto fora de 0–59 → erro de validação.

`[CONFIRMADO-DOC]` **Dois workflows distintos podem ter o mesmo horário; não há conflito entre
si.** — mesma seção.

> ⚠️ Isso responde a **outra** pergunta. Ver L1 abaixo: o silêncio é sobre **o mesmo** workflow
> se sobrepondo, não sobre dois workflows diferentes.

`[CONFIRMADO-DOC]` **Variável tipo Arquivo não funciona em cron.** *"Se o fluxo dispara via cron
(sem usuário no controle), variáveis tipo arquivo não funcionam (não tem ninguém pra fazer
upload)."* Alternativas declaradas: usar **URL/Path** apontando para bucket S3 ou pasta de rede,
ou disparar via API com `variable_values` pré-resolvidos.
— `guias-de-uso/variaveis-e-arquivos-no-runtime.md § Workflows agendados (cron)`

`[CONFIRMADO-DOC]` **`cron` não é exportável** para SQL/Python: está na lista de tipos que
devolvem **HTTP 422** na exportação (grupo *"Controle de fluxo"*, junto de `manual` e
`call_workflow`). Só o round-trip YAML preserva.
— `guias-de-uso/exportar-e-importar.md § Cobertura V1`

`[LACUNA]` **Nenhuma fonte declara o comportamento de falha do disparo agendado.** Se a execução
falha, o agendador tenta de novo? Registra? Notifica? Nada nas fontes.

## Exemplos

`[CONFIRMADO-MCP]` Do próprio contrato:

```json
{"cron_expression": "0 8 * * 1-5"}
```

`[CONFIRMADO-DOC]` Para o piloto de margem (a cada 5 minutos), a expressão gerada pelo preset é:

```json
{"cron_expression": "*/5 * * * *"}
```

`[VÍDEO]` O exemplo montado em aula: *"todo dia às 10 horas da manhã, que ele faça a inserção"* →
*"todos os dias, horário específico, às 10 da manhã (...) Ah não, só de segunda a sexta"*. O
instrutor liga o agendamento a um nó CSV e reforça a sequência salvar → publicar → produção.
— `m3-2A:19-23`

## Observações para o piloto de margem

1. **L1 — execução concorrente é o risco número um e continua aberta.** O piloto roda a cada
   5 minutos; se uma execução demorar 6, não se sabe o que acontece. `[LACUNA]` A doc de cron
   **não menciona** `max_instances`, `coalesce` nem `misfire_grace_time` do APScheduler, apesar de
   nomear o APScheduler explicitamente (`nos/agendamento-(cron).md § Expressão cron gerada`). A
   varredura por `concorren|simultân|sobreposi|max_instances|coalesce|misfire|lock` em todo o
   acervo bruto **não retorna nada sobre cron**. Falta descobrir: a plataforma serializa execuções
   do mesmo workflow ou permite sobreposição? Se permite, existe primitiva de trava? Não é
   inferível — é pergunta para o suporte Viasoft.

2. **L29 — quem é o usuário efetivo? Achado novo, ver abaixo.** Duas evidências independentes,
   nenhuma conclusiva:
   - `[CONFIRMADO-DOC]` A plataforma descreve o disparo agendado como **sem usuário**:
     *"Se o fluxo dispara via cron (**sem usuário no controle**)"*.
     — `guias-de-uso/variaveis-e-arquivos-no-runtime.md § Workflows agendados (cron)`
   - `[CONFIRMADO-DOC]` O gate de produção é implementado como checagem em **dois endpoints
     HTTP**: *"`_enforce_production_gate` em `/workflows/{id}/execute` e `/test`"*.
     — `documentacao-tecnica/controle-de-acesso.md § Checklist`

   `[INFERIDO]` Se o gate vive nesses dois endpoints e o disparo agendado é interno ao agendador
   (não passa por `/execute`), então **cron provavelmente não é submetido ao gate** — o fluxo
   agendado não falharia com `403` a cada 5 minutos. O inverso do risco original, e igualmente
   sério: significa que o agendamento é um **caminho que escreve em produção sem checagem de
   papel**. Publicar um fluxo agendado é, na prática, conceder execução em produção permanente.
   **Não confirmado** — inferência a partir da lista de endpoints. Ver L29.

3. **Piso de 5 minutos é da interface (L15).** Se o piloto precisar de intervalo menor, a
   expressão pode ser gravada direto — mas `[LACUNA]` sem saber se a API valida.

4. **`triggered_at` em UTC** não serve como marco de janela incremental sem conversão. E, de
   toda forma, **não é preciso usar `triggered_at` para isso**: `sql_database` tem
   `incremental_column` + `reprocess_window` nativos (L3, resolvida). O timestamp do cron serve
   para auditoria, não para recorte de dados.

5. **Não configure variável tipo Arquivo** em nenhum nó do fluxo agendado — quebra em silêncio no
   agendamento.

6. **`cron` não é exportável para SQL/Python.** Se o versionamento do piloto depender de
   exportação legível, o gatilho não vai junto — use YAML.
