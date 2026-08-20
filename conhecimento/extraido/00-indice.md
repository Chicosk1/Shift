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
| [lacunas.md](../lacunas.md) | **21 lacunas** ordenadas por impacto no piloto de margem. 4 já fechadas ou refutadas |

## A produzir na Fase 1

| Destino | Conteúdo | Fonte |
|---|---|---|
| `glossario.md` | Termos da plataforma: interface vs. API/MCP | Docs + aulas + MCP |
| `nos/<tipo>.md` | Um arquivo por nó — o que faz, quando usar, config, entradas, saídas, erros | `describe_node` + aulas + `guias-de-uso/nos/` |
| `procedimentos/<nome>.md` | Passo a passo imperativo de tarefas completas | Aulas (módulos 1, 2 e 4) |

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
