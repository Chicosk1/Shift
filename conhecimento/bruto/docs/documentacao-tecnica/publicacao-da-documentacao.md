---
title: Publicação da documentação
---

A documentação do Shift é **docs-as-code**: o conteúdo canônico são os arquivos
markdown em `docs/introducao`, `docs/users` e `docs/technical`, na raiz do
monorepo. Quem renderiza é o app `shift-docs` (Fumadocs/Next.js), servido em
`/docs` — o frontend só faz *rewrite* daquele path para o container.

**Push na `main` que toca documentação publica em produção sozinho, em poucos
minutos.** É o único gatilho automático do repositório; código continua com
build e deploy manuais.

## Onde editar

| O quê | Onde |
| --- | --- |
| Texto das páginas | `docs/<categoria>/**.md` na raiz |
| Ordem na sidebar | `pages` do `_meta.json` da categoria |
| Imagens | `docs/<categoria>/**/assets/` (referencie como `/docs/<categoria>/.../assets/x.png`) |
| Tema, layout, busca | `shift-docs/` |

⚠️ **Não edite `shift-docs/content/docs/`.** Aquela pasta é *saída* do
`shift-docs/scripts/sync-docs.mjs`, não fonte: ela é apagada e regerada a cada
`pnpm dev` e a cada build de imagem. Editar lá dá a pior combinação possível —
a mudança aparece na sua máquina e nunca chega em produção. O mesmo vale para
`shift-docs/public/docs/`, espelho das imagens. As duas são gitignored.

## O caminho até produção

```
docs/**  ou  shift-docs/**
        │
        │  push na main
        ▼
pipeline (automatico, condicionado ao changeset)
        │
        ├─ build   imagem shift-docs, tag = 12 chars do commit  (~2 min)
        │
        └─ deploy  ops/deploy.sh <tag> --only shift-docs
                        │
                        ▼
                grava DOCS_IMAGE_TAG no .env de producao
                recria SO o container shift-docs
```

Commit que não toca `docs/**` nem `shift-docs/**` faz o Bitbucket pular os dois
steps — o pipeline aparece na lista sem consumir build.

Para publicar sob demanda (sem ter mexido em doc no último commit): **Run
pipeline → docs**.

## Por que a doc tem canal próprio

O caminho normal publica 6 imagens em 20–45 min, e o `deploy-prod`
correspondente recria backend e worker, roda `alembic upgrade head` e mata
execução de fluxo em voo. Corrigir uma frase custava isso. O resultado prático
era previsível: a doc de produção ficava atrás da doc que roda em `localhost`.

Três peças fazem o isolamento, e as três precisam existir juntas:

1. **`DOCS_IMAGE_TAG`** — `shift-docs` é o único serviço com tag própria no
`docker-compose.prod.yml`. Sem isso, publicar doc exigiria mover a tag do
stack inteiro. O default é aninhado
(`${DOCS_IMAGE_TAG:-${IMAGE_TAG:-latest}}`), então quem não define a
variável mantém o comportamento antigo: a doc segue a tag do stack.

2. **Filtro no build** — `scripts/build-and-publish.sh shift-docs` builda só
essa imagem. Sem argumentos, builda as 6 (comportamento histórico).

3. **Escopo no deploy** — `ops/deploy.sh <tag> --only shift-docs` pula o
preflight (o container de doc não participa de execução de fluxo), **não**
sincroniza `compose`/`ops/` e restringe o portão de saúde ao escopo.

O `--only` não sincroniza o compose de propósito: um commit de doc carrega tudo
o que já está na `main`, inclusive mudança de infra que ninguém deployou. Copiar
o compose ali a deixaria dormente no arquivo, entrando em vigor no próximo
restart — dias depois, sem ninguém ligar as duas coisas. Publicar texto não
move infra.

## Como as duas tags convivem

| Deploy | `IMAGE_TAG` | `DOCS_IMAGE_TAG` |
| --- | --- | --- |
| `deploy-prod` (completo) | vira a tag nova | **também** vira a tag nova |
| `docs` (escopado) | intocada | vira a tag nova |

O deploy completo grava as **duas** de propósito. Se ele só mexesse em
`IMAGE_TAG`, a primeira publicação de doc criaria `DOCS_IMAGE_TAG` no `.env` e —
como ela tem precedência no compose — a doc ficaria pinada para sempre: todo
deploy de produto depois disso terminaria verde anunciando a tag nova enquanto
`/docs` seguiria numa imagem antiga, sem nada no log indicando a divergência.

Com as duas governadas, "deployei a tag T" volta a significar que **todo** o
produção roda T, e entre releases o canal de doc é livre para andar na frente.

## Pré-requisito de rollout (uma vez)

O compose que está **no servidor** precisa conhecer `DOCS_IMAGE_TAG`. Como o
modo `--only` não sincroniza compose, o primeiro rollout exige um
**`deploy-prod` completo**. Enquanto isso não acontecer, o deploy escopado falha
com mensagem explícita — em vez de terminar verde sem publicar nada.

## Categoria nova

Adicionar uma seção ao site exige **três** mudanças, e esquecer qualquer uma
delas quebra o build (de propósito):

1. `CATEGORIES` em `shift-docs/scripts/sync-docs.mjs`;

2. `COPY docs/<categoria> /docs/<categoria>` em `shift-docs/Dockerfile`;

3. `!docs/<categoria>` e `!docs/<categoria>/**` no `.dockerignore` da raiz.

Isto já custou uma seção inteira: `introducao` estava em `CATEGORIES` mas não
nas outras duas listas. O `sync-docs` avisava no log e seguia **verde**, e a
imagem de produção saía sem a seção — enquanto na máquina de quem escreve tudo
aparecia, porque lá o sync lê `docs/` direto. O sintoma em produção era pior que
um 404: `/docs/introducao/visao-geral` respondia **HTTP 200** com a página de
"não encontrado", e o `/docs` linkava para ela. Nenhum monitoramento por status
code veria isso. Hoje categoria ausente **aborta** o build.

## Rollback

Igual ao do produto: redeploye a tag anterior. Como o escopo de doc não roda
migration nem toca no resto do stack, o rollback é completo — não há efeito
colateral a investigar. `cat /opt/shift/.deploy/last-docs` diz que doc está no
ar (o `last` continua respondendo pela versão do produto).
