# Nó `inline_data` — Dados Embutidos

| | |
|---|---|
| **Tipo (MCP)** | `inline_data` |
| **Rótulo na interface** | Dados Embutidos |
| **Categoria** | `input` (grupo *Entradas*) |
| **Risco** | `read_only` `[CONFIRMADO-MCP]` |
| **Aliases aceitos** | `[LACUNA]` — o contrato não declara alias para este nó |
| **Sucessor consolidado** | — |

## O que faz

`[CONFIRMADO-MCP]` Materializa **dados estáticos embutidos no JSON do workflow** como tabela
DuckDB. *"Útil para tabelas de domínio pequenas, fixtures de teste e valores de referência."*
— `describe_node('inline_data')`

O ponto arquitetural: os dados **moram no próprio workflow**. Não há arquivo, upload, conexão nem
objeto externo — o conteúdo viaja com o fluxo, é versionado junto com ele e é editável no canvas.

## Quando usar

`[CONFIRMADO-MCP]`
- Inserir uma lista estática de valores para uso em lookup/join.
- Criar uma tabela de referência diretamente no canvas.
- Fornecer dados de teste sem necessidade de arquivo externo.

## Parâmetros de configuração

`[CONFIRMADO-MCP]` — `describe_node('inline_data')`

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `data` | array | **sim** | — | **Lista de dicts, dict único ou string JSON** com os dados |
| `output_field` | string | não | `data` | Nome do campo de saída |

`[CONFIRMADO-MCP]` O parâmetro `data` é declarado como tipo `array` mas aceita **três formas**:
lista de dicionários (o caso normal, uma linha por dict), **dicionário único** (uma linha só) e
**string JSON** (o JSON serializado em texto). É o nó de configuração mais enxuta do grupo de
entradas: dois parâmetros, um obrigatório.

## Entradas esperadas

Nenhuma tabular — é nó de entrada, e sem dependência externa nenhuma: não pede `url`, não pede
`connection_id`, não pede arquivo.

## Saídas produzidas

`[CONFIRMADO-MCP]` Uma **tabela DuckDB** no campo `output_field` (padrão `data`), com uma coluna
por chave dos dicionários fornecidos.

`[LACUNA]` **A tipagem das colunas não está declarada.** O contrato diz "materializa como tabela
DuckDB" mas não diz se o tipo é inferido do JSON (número JSON → coluna numérica), se tudo vira
texto, nem o que acontece quando a mesma chave tem tipos diferentes entre dicts.

`[LACUNA]` **Chaves desiguais entre dicts** — se um dict tem uma chave que outro não tem, o
contrato não diz se a coluna aparece com NULL, se o nó falha, ou se só as chaves do primeiro dict
contam. Não há parâmetro equivalente ao `null_padding` do `csv_input`.

## Erros comuns

`[LACUNA]` **O contrato deste nó não declara nenhuma armadilha.** É o único dos seis nós deste lote
sem seção "Armadilhas conhecidas" — os outros cinco têm de duas a três cada. Isso não significa que
não haja armadilhas: significa que o contrato não as documenta.

O que **não está declarado** e seria preciso saber antes de usar em produção:

`[LACUNA]` **Não há limite de tamanho declarado** para `data`. O contrato diz "tabelas de domínio
pequenas" e "valores de referência", o que sinaliza intenção de uso pequeno, mas **não fixa
número** — nem de linhas, nem de bytes, nem do JSON do workflow como um todo. Sem esse teto, não
se pode afirmar até onde o nó escala.

`[LACUNA]` **Não está declarado como se edita o `data`.** "Criar uma tabela de referência
diretamente no canvas" implica edição pela interface, mas o contrato não descreve o editor: se é
campo de texto com JSON cru, grade tipo planilha, ou upload convertido. Isso decide se um usuário
de negócio consegue manter a tabela sozinho.

`[LACUNA]` **Não está declarado o efeito no versionamento e no diff.** Os dados ficam no JSON do
workflow, então alterá-los altera o workflow — mas o contrato não diz se isso gera nova versão do
fluxo, se exige republicação, nem quem tem permissão para editar.

## Exemplo

`[CONFIRMADO-MCP]` Do próprio contrato — *"Como criar uma tabela de UFs diretamente no fluxo?"*:

```json
{"data": [{"uf": "SP", "nome": "São Paulo"}, {"uf": "RJ", "nome": "Rio de Janeiro"}]}
```

O exemplo é um **de-para de duas colunas** — exatamente a forma de uma tabela de domínio: chave de
lookup + valor.

## Observações

`[CONFIRMADO-MCP]` **Contraste com a base de dados interna.** Para guardar uma tabela de referência
dentro do Shift há duas rotas: `inline_data` (dados no JSON do fluxo, sem objeto separado) e a base
de dados interna (objeto próprio, criado à parte, teto de 200 mil linhas). O `inline_data` troca
capacidade por acoplamento: nada para criar, nada para manter fora do fluxo, versionado junto — em
troca de um volume que o contrato descreve apenas como "pequeno".

A escolha vira decisão de operação, não de tecnologia: **quem edita a tabela e com que
frequência**. Tabela que muda por decisão de negócio revisada raramente, com poucas linhas, cabe
embutida. Tabela que muda toda semana, ou que várias pessoas editam, ou que é lida por mais de um
fluxo, quer objeto próprio — porque o `inline_data` é **local ao fluxo**: `[LACUNA]` o contrato
não oferece nenhuma forma de compartilhar o mesmo `inline_data` entre workflows, o que implica
copiar e manter em paralelo.

`[CONFIRMADO-MCP]` O caso de uso "fixture de teste" é declarado explicitamente (*"Fornecer dados de
teste sem necessidade de arquivo externo"*). Vale para desenvolver um fluxo cuja fonte real é
lenta, caro de consultar, ou ainda não existe: troca-se a entrada real por um `inline_data` com
poucas linhas representativas.
