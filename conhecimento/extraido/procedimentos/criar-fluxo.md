# Procedimento: criar um fluxo e organizá-lo

**Fonte:** `m1:56-63` `[UI-OBSERVADA]` / `[VÍDEO]`, com `primeiros-passos.md §1`
`[CONFIRMADO-DOC]`.
**Pré-requisito:** `[CONFIRMADO-DOC]` estar dentro de um **projeto** — é ele que guarda os fluxos
(`primeiros-passos.md § O que você vai precisar`).

---

## Criar

1. **Vá ao menu Fluxos.**
   → Deve aparecer a listagem de fluxos do projeto, com filtros rápidos e busca.

2. **(Opcional) Crie uma pasta** para organizar.
   → Ao entrar na pasta, a listagem mostra só os fluxos dela.

3. **Clique em Novo Fluxo** e escolha **em branco** ou **a partir de um template**.
   → Só fluxos marcados como template aparecem na segunda opção.

4. **Informe o nome do fluxo.**

5. **Defina a visibilidade:** "só você vê" ou **"compartilhar com o time"**.
   → ⚠️ "Compartilhar com o time" significa **todos do espaço**, não um subconjunto (`m1:60`).

6. **(Opcional) Em Mais Opções**, informe descrição e tag.

7. **Confirme.**
   → Você cai direto no **canvas** — a tela onde os nós moram.

8. **(Opcional) Arraste o fluxo para dentro de uma pasta**, se criou fora dela.
   → Funciona por arraste direto na listagem.

---

## Organizar e encontrar depois

`[UI-OBSERVADA]` Recursos da listagem (`m1:56-58`):

- **Pastas** — arraste o fluxo para dentro.
- **Tags** — filtro por tag (ex.: tag `ConstruShow` traz tudo daquela vertical).
- **Filtros rápidos** — "em teste" e "em produção".
- **Busca textual** por nome.
- **Troca de visualização** da listagem.

Menu de contexto por fluxo: abrir, ver documentação, **exportar**, duplicar, **acesso e
visibilidade**, mover para pasta.

---

## Exportar e importar

`[VÍDEO]` O menu oferece exportar e importar, e o motivo declarado é justamente que **não existe
transferência de fluxo entre espaços** — se o ConstruShow desenvolveu algo que o Agro pode usar,
exporta-se de um lado e importa-se do outro. O instrutor diz que é **via JSON**. — `m1:58`

> ⚠️ **CONFLITO** — `guias-de-uso/exportar-e-importar.md` `[CONFIRMADO-DOC]` diz outra coisa:
> há **quatro** formatos de exportação (JSON canvas, SQL DuckDB, Python, YAML), mas
> **round-trip só em YAML**. O `Exportar JSON (canvas)` é descrito como *"formato legado,
> restaura o canvas"*, e SQL/Python são *"somente leitura — não há import equivalente"*.
> A importação é `POST /api/v1/workflows/import?workspace_id=<uuid>`, cria **draft** no workspace
> informado e descarta o `workflow_id` original.
>
> **Registro dos dois lados sem escolher.** Leitura provável: a aula é anterior aos formatos
> novos, ou "JSON" ali é o formato legado de canvas — que de fato importa, embora a doc
> recomende YAML. **Falta confirmar** qual dos dois o botão da interface usa hoje.

`[CONFIRMADO-DOC]` Restrições da exportação que importam ao versionamento:
`bulk_insert` e `cron` **não são exportáveis** para SQL/Python (HTTP 422 com lista de `node_id`),
e **fluxos com ciclo não são exportáveis**. Variáveis e conexões **não** são pré-resolvidas:
`{{vars.X}}` vira `${X}` e `connection_id` fica como TODO a substituir.
— `guias-de-uso/exportar-e-importar.md`

---

## Observações

`[CONFIRMADO-DOC]` Um fluxo existe como **instância** (dentro de um projeto, roda contra dados
reais) ou **template** (no espaço, sem cliente, serve de molde). Além de clonar, um projeto pode
**usar** um fluxo do espaço direto como sub-fluxo, sem cópia — assim uma correção no template
vale para todos de uma vez. — `conceitos.md § Fluxo`

`[LACUNA]` Não foi demonstrado **publicar** um fluxo nem alternar entre teste e produção, apesar
de os filtros "em teste"/"em produção" existirem na listagem. Isso importa porque
`nos/agendamento-(cron).md` `[CONFIRMADO-DOC]` afirma que **cron só dispara com o fluxo publicado
e em modo Produção**. Extração prevista no lote 4 (`m3p1`).
