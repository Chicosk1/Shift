# Índice da base de conhecimento

Estado em 2026-08-20, ao fim da **Fase 0**. A extração estruturada é a Fase 1 — as pastas
`nos/` e `procedimentos/` estão propositalmente vazias até lá.

---

## Validado contra o MCP (`[CONFIRMADO-MCP]`)

| Arquivo | Conteúdo |
|---|---|
| [mcp-ferramentas.md](../validacao/mcp-ferramentas.md) | As **48 ferramentas** do MCP com descrição e parâmetros, agrupadas por leitura / escrita / execução / build session. Inclui as pegadinhas de setup |
| [mcp-catalogo-nos.md](../validacao/mcp-catalogo-nos.md) | Os **63 tipos de nó** por categoria, com as pegadinhas declaradas nas próprias descrições |
| [divergencias.md](../validacao/divergencias.md) | 6 divergências doc↔MCP, o que foi confirmado, e o que existe só num lado |

## Pendências

| Arquivo | Conteúdo |
|---|---|
| [lacunas.md](../lacunas.md) | **28 lacunas** ordenadas por impacto no piloto de margem. 7 fechadas ou refutadas |

## ⚠️ Escopo de banco: **somente Oracle**

Definido pelo usuário em 2026-08-20: as automações do ERP rodam **exclusivamente sobre Oracle**.
A base de conhecimento registra o que as fontes dizem sobre os outros bancos (é fato, e fato não
se apaga), mas **as skills devem cobrir só Oracle** — skill inchada com caminhos que nunca serão
usados perde eficácia no acionamento.

Consequências para a extração:
- **Sobem de prioridade** os documentos Oracle-específicos que pareciam periféricos:
  `toleracia-a-erro-por-linha-na-borda-design.md` (usa `LOG ERRORS` do Oracle),
  `upsert-rapido-staging-+-merge-set-based-(design).md` (usa GTT),
  `extracao-na-borda-design.md` (**declaradamente só Oracle**). Os recursos de borda **se
  aplicam** a este caso.
- **Risco a verificar:** `planejamento-dlt-(extracao-insercao).md` registra **bug conhecido de
  Oracle + dlt**, com Oracle em fallback SQLAlchemy.
- **Descem de prioridade:** `deploy-firebird.md`, os objetivos SQL Server de
  `comando_para_maquina_local`, e as partes de `m2p2` que criam conexão SQL Server.

## Extraído (Fase 1)

### Glossário
[glossario.md](glossario.md) — termos da plataforma com definição e a coluna **interface vs.
MCP/API**. Cobre a hierarquia completa, as três peças (conexão, fluxo, nó), categorias de nó e
termos de execução. **Lote 1.**

### Nós — `nos/`
Nome do arquivo = **tipo canônico do MCP** (o identificador que `describe_node` aceita).

| Arquivo | Tipo | Categoria | Risco | Lote |
|---|---|---|---|---|
| [manual.md](nos/manual.md) | `manual` | trigger | read_only | 1 |
| [csv-input.md](nos/csv-input.md) | `csv_input` | input | read_only | 1 |
| [mapper.md](nos/mapper.md) | `mapper` | transform | read_only | 1 |
| [bulk-insert.md](nos/bulk-insert.md) | `bulk_insert` | database | **write** | 1 |

*4 de 63 nós catalogados.* Índice completo dos 63 em
[mcp-catalogo-nos.md](../validacao/mcp-catalogo-nos.md).

### Procedimentos — `procedimentos/`

| Arquivo | Cobre | Lote |
|---|---|---|
| [criar-espaco.md](procedimentos/criar-espaco.md) | Criar e configurar Espaço (sistemas, cor, ícone, favorito) | 1 |
| [criar-fluxo.md](procedimentos/criar-fluxo.md) | Criar fluxo, organizar por pasta/tag, exportar e importar | 1 |
| [primeiro-fluxo-csv-para-tabela.md](procedimentos/primeiro-fluxo-csv-para-tabela.md) | Fluxo ponta a ponta: Manual → CSV → Mapeamento → Inserção em Massa | 1 |

## A produzir nos lotes seguintes

| Destino | Conteúdo | Lote previsto |
|---|---|---|
| `procedimentos/conceder-acesso.md` | Dar acesso de usuário ao espaço | 2 |
| `procedimentos/configurar-conexao-*.md` | Conexão direta, TNS, relay, por conjunto de arquivos | 2 e 3 |
| `procedimentos/publicar-e-agendar.md` | Publicar, alternar teste/produção, agendar | 4 |
| `procedimentos/monitorar-execucao.md` | Ler execução, rejeições, dead-letter | 4 e 12 |
| `nos/*.md` | Os 59 nós restantes | 4 a 11 |

---

## Material bruto

- `bruto/transcricoes/` — **32 arquivos, 4.903 linhas.** Módulos 1 a 4. Sem timestamp:
  proveniência por **arquivo + número de linha**; marcações de cena no formato `(AÇÃO: …)`
- `bruto/docs/` — **40 documentos oficiais.** Introdução (3), guias de uso (4), páginas de nó
  (14), documentação técnica (19)

**Cuidado com 6 documentos técnicos** que são proposta ou design, não plataforma em produção —
ver `lacunas.md` L17. Nunca marcar como `[CONFIRMADO-DOC]`.

## Marcadores de confiança

| Marcador | Significado |
|---|---|
| `[CONFIRMADO-MCP]` | Verificado contra as ferramentas reais do MCP. **Vence qualquer outra fonte** |
| `[CONFIRMADO-DOC]` | Explícito em documentação de estado corrente |
| `[UI-OBSERVADA]` | Rótulo literal de campo lido na tela da aula |
| `[VÍDEO]` | Afirmação narrada em aula, não confirmada |
| `[INFERIDO]` | Dedução, não observado |
| `[LACUNA]` | Sabe-se que existe, não se sabe como funciona |

Só `[CONFIRMADO-*]` pode virar instrução imperativa numa SKILL.md.

## Ordem de processamento da Fase 1

**14 lotes**, de no máximo 4 arquivos, documento oficial antes da aula dentro de cada tema.
Detalhe no plano-mestre; lotes 6b e 6c são o módulo 4.
