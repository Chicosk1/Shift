---
title: Primeiros passos
description: Construa e execute seu primeiro fluxo no Shift — de um arquivo CSV para uma tabela Postgres.
---

Vamos montar o fluxo mais comum do Shift de ponta a ponta: **ler um CSV e gravar numa tabela do banco**. Leva uns 10 minutos e cobre o ciclo inteiro — ler, (opcionalmente) ajustar e gravar.

> Antes de começar, vale entender o [vocabulário](https://shift.viasoftcloud.com.br/docs/introducao/conceitos) (workspace, projeto, conexão, fluxo, nó). São 5 ideias rápidas.

## O que você vai precisar

- Estar dentro de um **projeto** (é ele que guarda os fluxos).

- Uma **conexão** com o banco de destino (Postgres, neste exemplo) já cadastrada no workspace. Conexão é credencial guardada e reutilizável — você cadastra uma vez.

- O **arquivo CSV** que vai subir. Se não tiver um à mão, qualquer planilha exportada como CSV serve.

- Uma **tabela de destino** já existente no banco, com colunas compatíveis com o CSV.

## 1. Crie um fluxo

Dentro do projeto, crie um fluxo novo e dê um nome (ex: `importa-clientes`). Você cai no **canvas** — a tela onde os nós moram.

## 2. Leia o CSV

Abra a **biblioteca de nós** e adicione o nó **CSV** (grupo *Entradas*). No painel de configuração:

- **Arquivo** — faça o upload do seu CSV (ou aponte uma URL/caminho).

- **Delimitador** — `,` por padrão. Se o arquivo usa `;`, troque aqui.

- **Cabeçalho** — deixe ligado se a primeira linha tem os nomes das colunas.

Rode o **preview** do nó para conferir que as linhas aparecem como esperado. Se o cliente costuma errar nomes de coluna, vincule um [Modelo de Entrada](https://shift.viasoftcloud.com.br/docs/docs/users/modelos-entrada): o Shift valida o arquivo **antes** de processar e falha com uma mensagem clara em vez de um erro críptico lá na frente.

## 3. (Opcional) Alinhe as colunas

Se os nomes das colunas do CSV não batem com os da tabela de destino, adicione um nó **Mapeamento** (grupo *Transformação*) entre o CSV e a saída. Nele você diz, por exemplo, que a coluna `cnpjcpf` do arquivo vai para `cnpj_cpf` da tabela. Pode pular este passo se os nomes já coincidem.

## 4. Grave na tabela

Adicione o nó **Inserção em Massa** (grupo *Banco de Dados*). Configure:

- **Conexão** — escolha a conexão Postgres do pré-requisito.

- **Tabela de destino** — o nome da tabela que vai receber as linhas.

- **Mapeamento de colunas** — quais colunas da entrada vão para quais colunas da tabela.

- **Estratégia de carga** — comece com *append* (`append_fast`, o padrão: só insere). Há ainda `append_safe` (mesma coisa, mas desfaz tudo se uma linha falhar), `upsert` e `insert_if_not_exists`, úteis para cargas que rodam repetidas vezes sem duplicar — essas duas últimas exigem indicar qual coluna identifica um registro único (ex.: `cnpj_cpf`).

> **Opcional — sem banco de dados à mão?** Troque o nó por **Base de Dados Interna (Escrita)** (mesmo grupo *Banco de Dados*). Em vez de uma conexão externa, ele grava direto numa **Base de Dados** do próprio Shift — crie uma em *Bases de Dados* (menu do espaço), com o nome que quiser e as colunas do seu CSV. Configure:
>
>
>
> - **Base de Dados** — a que você acabou de criar.
>
> - **Modo** — comece com *insert* (só anexa linhas). Há também *upsert*, *update*, *delete* e *replace*, que casam por uma coluna-chave (ex.: `cnpj_cpf`).
>
>
> Serve bem para testar o fluxo sem depender de conexão nenhuma, mas é pensada para cadastros pequenos (teto de 200 mil linhas, porque vive no banco da própria plataforma) — para carga de produção real, uma conexão com banco externo continua sendo o caminho certo.

## 5. Conecte os nós

Arraste da **bolinha de saída** de um nó até a **bolinha de entrada** do próximo, nesta ordem: **CSV → (Mapeamento) → Inserção em Massa**. O dado flui por essas conexões.

## 6. Execute

Rode o fluxo. Ao terminar, o painel de execução mostra o **`row_count`** — quantas linhas foram lidas e gravadas. Se algo falhar, a mensagem aponta o nó e o motivo (coluna ausente, tipo incompatível, etc.).

Pronto — você acabou de fazer sua primeira carga. 🎉

## E agora?

- Veja o catálogo completo de caixinhas na seção **Nós** — o [CSV](https://shift.viasoftcloud.com.br/docs/docs/users/nodes/csv) que você usou tem mais opções (encoding, retentativa, limite de linhas…).

- Parametrize o fluxo com [Variáveis e arquivos](https://shift.viasoftcloud.com.br/docs/docs/users/variaveis-arquivos), para o arquivo mudar a cada execução.

- Garanta a qualidade do que entra com [Modelos de Entrada](https://shift.viasoftcloud.com.br/docs/docs/users/modelos-entrada).
