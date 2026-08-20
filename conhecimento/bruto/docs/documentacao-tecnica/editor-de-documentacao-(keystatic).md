---
title: Editor de Documentação (Keystatic)
---

> ⛔ **Este caminho não funciona hoje, e não é o fluxo de publicação.** Para
> publicar documentação, veja **[Publicação da
> documentação](https://shift.viasoftcloud.com.br/docs/docs/technical/publicacao-documentacao)** — push na `main` que
> toca `docs/**` já publica sozinho.
>
> O bloqueio é de plataforma, não de configuração: **o repositório do Shift está
> no Bitbucket** (`bitbucket.org/viasoftnimitz/shift`), e o Keystatic só suporta
> storage `github` ou `local`. Não existe valor de `KEYSTATIC_STORAGE` que
> commite no Bitbucket, então todo o checklist de GitHub App abaixo é
> inaplicável enquanto o repositório for esse. O modo `local` também não
> alcança a doc canônica (o Keystatic recusa `..`, e `docs/` é pasta-irmã do
> app).
>
> O documento fica como registro do desenho e do que seria necessário. Se a meta
> for "quem escreve doc não mexe em git", o caminho passa por outra ferramenta
> (com suporte a Bitbucket) ou por migrar de host — decisão em aberto.

A documentação do Shift é **docs-as-code**: os arquivos markdown vivem em
`docs/introducao`, `docs/users` e `docs/technical` no repositório. Para edição
amigável (sem mexer em arquivos direto), a ideia era o **Keystatic** — um editor
visual servido em `/keystatic` dentro do app, que gravaria as alterações como
**commits no repositório**. O site público (`/docs`) é então reconstruído pela
CI.

## Como funciona

```
/keystatic (editor, autenticado)  ──commit/PR──►  repositório (docs/*)
                                                        │
                                                   CI rebuilda
                                                        ▼
                                              imagem shift-docs  ──►  /docs
```

- **Edição** acontece no `shift-frontend` (rota `/keystatic`).

- O conteúdo canônico continua em `docs/users` e `docs/technical`.

- **Renderização** é o site `shift-docs` em `/docs`.

## Modos de armazenamento

Controlado pela env `KEYSTATIC_STORAGE`:

| Valor | Uso | Comportamento |
| --- | --- | --- |
| `github` | **Produção** | Grava via API do GitHub (commits/PR). Autoriza por GitHub App. |
| ausente/outro | Dev local | Grava no checkout local (requer rodar a partir da raiz do repo). |

## Setup da GitHub App (produção) — checklist

> Passo de infra, feito uma vez. Necessário para o modo autorizado.

1. **Criar a GitHub App** no org do repositório (`Settings → Developer settings → GitHub Apps → New`). O jeito mais simples é deixar o próprio Keystatic guiar:
acesse `/keystatic` com `KEYSTATIC_STORAGE=github` e siga o fluxo de
"set up GitHub App", que pré-preenche as permissões.

2. **Permissões da App**: `Contents: Read and write` e `Pull requests: Read and write` no repositório `viasoft/shift-project` (ajustar owner/repo conforme o
real). Instalar a App **nesse repositório**.

3. **Callback URL**: `https://shift.viasoftcloud.com.br/api/keystatic/github/oauth/callback`.

4. **Coletar credenciais** e setar as envs do `shift-frontend`:

```
KEYSTATIC_STORAGE=github
KEYSTATIC_GITHUB_OWNER=viasoft
KEYSTATIC_GITHUB_REPO=shift-project
KEYSTATIC_GITHUB_CLIENT_ID=<client id da App>
KEYSTATIC_GITHUB_CLIENT_SECRET=<client secret da App>
KEYSTATIC_SECRET=<string aleatória; ex: openssl rand -hex 32>
NEXT_PUBLIC_KEYSTATIC_GITHUB_APP_SLUG=<slug da App>
```

5. **Autorização de quem edita**: só usuários com acesso de escrita ao
repositório (e que instalaram/autorizaram a App) conseguem commitar. Para
autores sem GitHub, criar conta e dar acesso ao repo, ou centralizar num grupo
de mantenedores.

> ⚠️ **Compatibilidade:** o `@keystatic/next` tem como alvo o Next 15; o projeto
> está no Next 16. O modo local apresentou um 400 na API de filesystem. O modo
> `github` usa outro caminho de código — validar ao concluir o setup. Se houver
> incompatibilidade, avaliar TinaCMS ou aguardar suporte oficial a Next 16.

## Fluxo do autor (uso diário)

1. Acessa `https://shift.viasoftcloud.com.br/keystatic` (logado).

2. Escolhe **Guias de uso** ou **Documentação técnica**.

3. Edita no editor visual (título, ordem na sidebar, conteúdo).

4. Salva → vira commit/PR no repositório.

5. A CI reconstrói o site; a alteração aparece em `/docs`.

## Observações de conteúdo

- O Keystatic grava conteúdo como **MDX** (`.mdx`). Os docs atuais são `.md`;
na adoção, converter `.md` → `.mdx` (o Fumadocs lê os dois nativamente) e
validar que tabelas, HTML e blocos de código sobrevivem ao round-trip do
editor.

- A **ordem na sidebar** usa o campo `order` do frontmatter.
