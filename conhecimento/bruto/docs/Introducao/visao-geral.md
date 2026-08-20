---
title: Visão geral
description: O que é o Shift, para que serve e como esta documentação está organizada.
---

O **Shift** é a plataforma de **integração, migração e automação de dados**. Em
vez de escrever e manter scripts soltos, você monta **fluxos visuais**: um
diagrama de caixinhas (os *nós*) que leem dados de uma origem, transformam e
gravam num destino. Se fossemos resumir o que é o Shift em uma frase seria:

> Shift é uma plataforma de automação sem código através de fluxos, ond o mesmo
> pode ser disparado por uma **Pessoa**, por um **Agente de IA** ou por um
> **Evento** de sistema.

Casos típicos:

- **Migração de cliente** — pegar a base de um sistema antigo (SQL
Server, Firebird…) e carregar no novo, ajustando colunas e formatos no caminho.

- **Carga recorrente** — todo dia às 6h, ler um CSV que o cliente deposita,
validar e inserir numa tabela.

- **Integração entre sistemas** — receber um webhook de um sistema e refletir o
dado em outro, na hora.

- **Captura manual de dados** — um formulário onde alguém preenche informação
à mão, que vira dado disponível para o fluxo usar — sem depender de arquivo
ou banco de origem.

- **Relatórios e avisos** — montar um PDF a partir dos dados e mandar por
e-mail ou WhatsApp ao final do fluxo, sem intervenção manual.

- **Análises e decisão** — aplicar cálculos estatísticos prontos (score de
crédito, detecção de anomalia, previsão, RFM…) sobre os dados que passam
pelo fluxo.

- **Automação de processos** — qualquer processo repetitivo, com etapas de
lógica (condições, loops, sub-fluxos) e um caminho para tratar exceções que
precisam de atenção humana.

## Quando usar (e quando não)

**Use o Shift quando** o trabalho é mover e transformar dados entre origens e
destinos de forma repetível, auditável e sem depender de alguém rodar um script
na mão.

**Provavelmente não é o Shift quando** você precisa de uma aplicação interativa
com regra de negócio por requisição (isso é backend de produto), ou de um BI
para explorar dados visualmente (isso é ferramenta de dashboard). O Shift
*alimenta* esses sistemas; ele não os substitui.

## Como esta documentação está organizada

- **Introdução** (você está aqui) — o panorama. Siga para os
[Conceitos](https://shift.viasoftcloud.com.br/docs/introducao/conceitos) e depois coloque a mão na massa em
[Primeiros passos](https://shift.viasoftcloud.com.br/docs/introducao/primeiros-passos).

- **Guias de uso** — como fazer as coisas: conexões, modelos de entrada,
variáveis, importar e exportar.

- **Nós** — a referência de cada caixinha disponível no fluxo.

- **Documentação técnica** — arquitetura, deploy e controle de acesso, para quem
opera a plataforma.

Use a **busca** (no topo) para pular direto a qualquer assunto.

## Por onde começar

1. Entenda o vocabulário em [Conceitos](https://shift.viasoftcloud.com.br/docs/introducao/conceitos) — são 5 ideias.

2. Construa seu primeiro fluxo em [Primeiros passos](https://shift.viasoftcloud.com.br/docs/introducao/primeiros-passos).
