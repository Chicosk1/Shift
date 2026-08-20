# Nó `workflow_output` — Saída do Fluxo (Saída do Sub-workflow)

| | |
|---|---|
| **Tipo (MCP)** | `workflow_output` |
| **Rótulo na interface** | Saída do Fluxo / Saída do subfluxo |
| **Categoria** | `subflow` `[CONFIRMADO-MCP]` |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Sucessor consolidado** | — (nenhum sucessor declarado nas fontes) |

> `[LACUNA]` **Este nó não tem página de documentação de uso.** `guias-de-uso/nos/` contém 14
> arquivos e **nenhum** para `workflow_output` (nem para `call_workflow`). As únicas fontes são o
> contrato do MCP, a tabela de relacionamento em `nos/entrada-do-fluxo.md § Relacionamento com
> outros nós`, e a aula `m3-2A`. Isso deixa o nó mais fraco em `[CONFIRMADO-DOC]` do que os
> gatilhos — e ele é obrigatório para o subfluxo devolver qualquer coisa.

## O que faz

`[CONFIRMADO-MCP]` Ponto de saída interno de um sub-workflow. **Captura campos do contexto** e os
empacota como resultado retornado ao nó `call_workflow` pai.
— `describe_node('workflow_output')`

`[CONFIRMADO-DOC]` *"Empacota os resultados que serão devolvidos ao chamador."*
— `nos/entrada-do-fluxo.md § Relacionamento com outros nós`

`[VÍDEO]` A aula define o par pela forma: *"quando eu chamo um fluxo, ele é como se fosse um nó.
Só que ele é um nó com várias lógicas dentro. Então ele tem a entrada, esse cara aqui, e ele vai
ter uma saída."* E: *"quando é um subfluxo, por padrão, ele sempre vai ter essa característica
aqui. Ele vai ter uma entrada e vai ter uma saída."* — `m3-2A:63`, `m3-2A:65`

## Quando usar

`[CONFIRMADO-MCP]`
- Publicar resultados do sub-workflow para o fluxo pai.
- Declarar os outputs que o sub-workflow produz.

`[VÍDEO]` O critério prático, dito no fecho da aula:

> *"Se você vai usar aqueles dados que gerou dentro desse subfluxo aqui em alguma outra parte,
> você **sempre** tem que definir uma saída."* — `m3-2A:109`

E o corolário — sem saída, o subfluxo funciona, mas devolve vazio: *"Por que que ele não trouxe
nada aqui? Porque lá dentro do nosso subfluxo a gente não definiu uma saída. Então **quando não
tem uma saída ele só executa**."* — `m3-2A:98`

> **Isto é uma armadilha de silêncio, não um erro.** O subfluxo roda, grava no banco, completa com
> sucesso — e o pai recebe nada. Não há mensagem de erro. Foi exatamente o que aconteceu em aula:
> os 300 funcionários **foram inseridos** e ainda assim o retorno veio vazio (`m3-2A:95-98`).

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('workflow_output')`

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mapping` | object | **sim** | Mapeamento `nome -> expressão template` dos campos a capturar |

**É o único parâmetro, e é obrigatório** — coerente com o fato de que um nó de saída sem
mapeamento não teria o que devolver.

> Note a assimetria com o par: `workflow_input` tem `output_field` **opcional** (padrão `data`);
> `workflow_output` tem `mapping` **obrigatório**. A entrada tem default útil; a saída não pode ter.

`[UI-OBSERVADA]` Na tela, o `mapping` aparece como a pergunta "de onde vêm esses dados": *"Agora
eu posso discernir aqui: da onde que vai vir esses dados aí? Então ele vai vir do nosso último nó
ali que é o insert."* — `m3-2A:102`

`[UI-OBSERVADA]` E há **casamento por nome/descrição** entre o schema declarado e a origem:
*"Daí ele vai fazer por descrição ali a combinação do que que ele vai mandar."* — `m3-2A:103`

### O esquema de output mora no fluxo, não no nó

`[UI-OBSERVADA]` Como na entrada, o **contrato** de saída é declarado no painel de **Schema de
IO do fluxo**, não na configuração do nó: *"o que faltou, na verdade, é a gente vir aqui e definir
um **esquema de output**"*. — `m3-2A:100`

`[UI-OBSERVADA]` Passos observados (`m3-2A:100-101`):
1. Declarar o output no schema de IO — na aula, uma **tabela de referência** chamada
   `funcionarios_cadastrados`, com `ID` (inteiro) e `nome` (string), ambos marcados
   **obrigatórios**.
2. Recarregar o mock de teste e **rodar em modo teste** para *"carregar o esquema completo"* — sem
   isso, o nó de saída não conhece as colunas disponíveis para mapear.
3. Só então mapear a origem no nó.

> **Ordem que importa:** declarar schema → executar em teste para popular os esquemas → mapear a
> saída. O instrutor faz isso duas vezes na aula (`m3-2A:92`, `m3-2A:101`), inclusive desativando
> e reativando nós só para carregar esquema.

`[INFERIDO]` A mesma leitura de `workflow_input` se aplica: o esquema é propriedade do **workflow**
(a superfície do MCP expõe `pending_set_io_schema` como operação de fluxo). **Ferramenta de
escrita, não executada.**

## Entradas esperadas

O resultado do nó anterior do subfluxo — tipicamente o último nó útil da cadeia.

`[VÍDEO]` Em aula, a origem mapeada foi o nó de **inserção**: *"ele vai vir do nosso último nó ali
que é o insert (...) será devolvida ao insert"*. — `m3-2A:102`

`[LACUNA]` Não se sabe se o nó aceita **múltiplas entradas** no canvas, nem se o `mapping` pode
referenciar nós que **não** são o predecessor direto (o formato `nome -> expressão template`
sugere que sim, via `upstream_results`, mas nenhuma fonte confirma). A aula tenta algo nessa
direção e desiste: *"Eu poderia até deixar aqui que ele vai retornar todos, se eu não me engano.
Não. Não, tá certo, tem que ser aqui."* — `m3-2A:103`

`[LACUNA]` Também não se sabe se um subfluxo pode ter **mais de um** nó de saída. A restrição de
"apenas um" é declarada só para `workflow_input`
(`nos/entrada-do-fluxo.md § Limites e guardrails`); nada é dito sobre a saída.

## Saídas produzidas

`[CONFIRMADO-MCP]` Os campos capturados pelo `mapping`, entregues ao nó `call_workflow` do pai.

`[CONFIRMADO-MCP]` No pai, esse resultado chega no campo definido por `output_field` do
`call_workflow` — **padrão `workflow_result`**. — `describe_node('call_workflow')`

`[VÍDEO]` Confirmação de ponta a ponta: depois de definir a saída, *"vamos executar em produção.
Ó, agora ele retornou os dados, tá vendo?"* — e os dados retornados foram consumidos por um nó
**Exportar Excel** no fluxo pai. — `m3-2A:107-108`

## Erros comuns

`[VÍDEO]` **Saída não definida → retorno vazio, sem erro.** O caso principal, descrito acima.
— `m3-2A:98`

`[UI-OBSERVADA]` **Mapear antes de carregar o esquema** → o nó não tem colunas para escolher. A
aula resolve rodando em teste primeiro. — `m3-2A:101`

`[LACUNA]` **Nenhum erro é declarado no contrato do MCP.** Falta descobrir:
- O que acontece se o `mapping` referenciar um campo que não existe no contexto.
- O que acontece se o resultado **não casar** com o esquema de output declarado (campo obrigatório
  ausente, tipo incompatível) — há validação, ou passa? Por analogia, `workflow_input` tem erro
  declarado para parâmetro obrigatório ausente, mas ele estoura **no `call_workflow` do pai**
  (`nos/entrada-do-fluxo.md § Limites e guardrails`); nada equivalente está documentado para a
  saída.
- Se o teto de **1.000 linhas** da rehidratação (documentado para a **entrada**) também se aplica
  ao **retorno** do subfluxo para o pai. **Isto é importante e não está respondido em nenhuma
  fonte.**

## Exemplo

`[CONFIRMADO-MCP]` O contrato não traz exemplo próprio para este nó. Forma do parâmetro:

```json
{"mapping": {"nome_do_campo": "{{upstream_results.<nodeId>.coluna}}"}}
```

`[INFERIDO]` A sintaxe de template é a mesma usada em `call_workflow.input_mapping`, que o contrato
exemplifica como `{"pedido_id": "{{upstream_results.node_xyz.pedido_id}}"}`
`[CONFIRMADO-MCP]`. Não há exemplo confirmado de `workflow_output.mapping` em nenhuma fonte —
`[LACUNA]`.

`[VÍDEO]` O exemplo da aula, em prosa: no subfluxo `Funcionarios Core`, esquema de output declarado
como tabela de referência `funcionarios_cadastrados` com `ID` (inteiro, obrigatório) e `nome`
(string, obrigatório); o nó de Saída do Fluxo mapeia essas colunas a partir do nó de inserção.
O pai (`Inserir Funcionários`) recebe o retorno e o encaminha para um Exportar Excel.
— `m3-2A:100-108`

## Observações para o piloto de margem

1. **A trilha de auditoria do piloto depende deste nó.** Se o cálculo de preço virar subfluxo, é o
   `workflow_output` que devolve ao pai o que foi alterado — insumo direto da auditoria pedida no
   plano (§5). Combina com `returning_columns` do `bulk_insert`, que captura o valor efetivamente
   gravado (ver `nos/bulk-insert.md`). Mas **onde persistir essa auditoria continua sendo L28.**

2. **`mapping` obrigatório é uma proteção acidental.** Como o nó não pode existir sem mapeamento, o
   risco não é esquecer de mapear — é **não colocar o nó**. Item de checklist: *todo subfluxo do
   piloto tem nó de Saída?*

3. **`[LACUNA]` O limite de 1.000 linhas no caminho de volta é uma pergunta aberta com impacto
   direto.** A doc só o declara para a rehidratação da **entrada**. Se valer também na volta, um
   subfluxo que devolva a lista de itens reprecificados trunca em silêncio — e o relatório de
   auditoria ficaria incompleto exatamente nos lotes grandes, que são os que importam. Falta
   descobrir. Enquanto não se souber: **devolva agregados** (contagens, somas, ids de lote), não
   linhas.

4. **Consequência de projeto:** o par entrada/saída é bom para devolver **decisão e resumo**, não
   para transportar **volume**. Isso reforça a recomendação de `nos/workflow-input.md`: manter o
   caminho dos dados dentro de um fluxo só.
