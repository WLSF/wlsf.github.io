---
layout: post
title: "Pão de Queijo com Geleia"
date: 2024-08-21 09:58:00
categories: C/C++ IDA CheatEngine WatchDogs2 Hacking
---

Importante frizar: A ideia deste texto é mostrar um pouco da minha visão com engenharia reversa para novatos.

Tentarei manter o mais breve possível, infelizmente não cobrirei o passo a passo de como as ferramentas funcionam aqui.


# História e contexto

Se você não conhece e/ou nunca jogou, [Watch Dogs 2](https://en.wikipedia.org/wiki/Watch_Dogs_2) é o GTA que deu errado, feito pela Ubisoft. (brinks, o jogo é bom)

O contexto e o conceito do jogo envolvem personagens que são altamente revoltados com o governo e as iniciativas privadas de controle de dados. Ou seja, vc é um hacker que tenta se rebelar contra o sistema.

O jogo veio em 2016 e possui modo online. O que significa isso? se possui modo online e é um jogo relativamente novo, obviamente que vai possuir um sistema de anti-trapaças, mecanismos de ofuscação de código e até mesmo mecanismos que chamamos de anti-adulteração pra garantir que o jogo não será crackeado/pirateado.

# A Mentalidade por trás da Engenharia Reversa

Ao começar a estudar o jogo, eu consigo enumerar meus passos da seguinte maneira:

1. Analisar todos arquivos que vieram no download do jogo (no meu caso foi pela Steam)

Ter uma boa noção de todo os arquivos que vem disponibilizado no jogo te ajuda a ter uma perspectiva inicial, além de que, na medida que você vai se aprofundando no jogo, isso te permite entender melhor sobre o próprio jogo.

O Watch Dogs 2 da Steam não possui muita coisa.

-- bin\                             - bibliotecas (.dll) essenciais do jogo
-- data_win64\                      - arquivos que são utilizados para definir o mundo/missões/personagens etc...
-- EasyAntiCheat\                   - relacionado ao AntiCheat que o jogo usa no modo online
-- support\                         - sim, eu só ignorei essa pasta...
-- EAC (.exe)                       - executável
-- UbisoftConnectInstaller (.exe)   - executável tbm, irrelevante agora

2. Rodar o jogo pela primeira vez, fazer uma missão, pegar a vibe, deixar a vida me levar...

Eu nunca tinha jogado esse treco, nada melhor do que simplesmente abrir o jogo e entender como funciona.

![watch_dogs2.exe](/assets/game.jpg)

3. Passamos da fase de unboxing do jogo, agora meu primeiro teste precisa ser feito com o CheatEngine.

Meu primeiro passo favorito: aplicar um depurador em tempo real na aplicação rodando, isso vai te dar uma perspectiva do quão cabuloso o jogo é (ou não). No meu caso, eu uso bastante CheatEngine ou o x64dbg (falecido ollydbg).

Nota importante: Você sempre vai querer iniciar a sua depuração colocando o jogo em modo janela, assim, você facilitará sua vida ao invés de ter que ficar fazendo alt tab em um jogo fullscreen. Principalmente em jogos antigos, as versões antigas do directx e dos jogos pode não suportar certas resoluções e isso pode bugar todo seu monitor durante a sua interação com o jogo e o depurador.

Olhando os arquivos e a memória RAM enquanto o jogo roda, já consigo ver que eles usam um sistema de anti-trapaças comercial, mais conhecido como EAC (EasyAntiCheat). Isso faz com que o jogo não se comporte bem com um depuerador, eles possuem maneiras de te enganar.

Se o jogo possui modo online e modo história (offline), isso significa que o sistema anti-trapaças pode ser desativado, tendo em vista que, no modo offline ele seria inútil. (Mesmo esquema do GTA V)

Pra desativar o eac no modo história só precisei adicionar a opção '-eac_launcher' na inicialização do jogo, agora posso brincar a vontade.

![eac_launcher](/assets/eac_launcher.jpg)

Com o depurador plugado no jogo, abro o jogo e vou direto procurar o endereço de memória que referência o valor de dinheiro que o personagem possui. Dinheiro costuma ser algo visível e universal, maioria dos jogos vai usar o mesmo dado como 4 bytes decimal e a representação vai ser a mesma que o jogo renderiza, isso te permite fazer um scan com o depurador pra encontrar o valor de forma fácil.

![cheat engine](/assets/cheatengine.jpg)


Esse processo é bem manual, consiste em você ficar escaneando a memória do jogo enquanto efetua alterações no dinheiro do personagem, com a finalidade de encontrar o endereço. Uma vez com este endereço, eu posso simplesmente começar a aplicar breakpoints no jogo e encontrar as funcionalidades que fazem uso dessa área de memória.

Enquanto usando um depurador em tempo real no jogo, você precisará estar familiarizado com assembly e o conjunto de instruções do seu processador. Isso não significa que só porque o seu pc é x64 o jogo irá rodar nesse conjunto, muitos jogos fazem proveito de funcionalidades e coisas da arquitetura x86 ainda.

Muitas vezes utilizando os registradores de forma compacta, ao invés de fazer uso do seu RCX, ele irá utilizar o ECX, principalmente em funções do tipo __fastcall onde os primeiros 2 argumentos costumam ser passados para a função pelo ECX e o EDX, nessa ordem.

4. Partindo para análise estática do jogo

Usando ferramentas como IDA Pro / Ghidra você consegue ler o binário sem executar e ter acesso ao código assembly referente e até mesmo decompilar este código para C++.

A parte de análise estática é onde eu consigo tirar proveitos das seguintes coisas:

- Entender como o jogo é iniciado, quais tipos de dados estáticos e quais bibliotecas ele usa, as suas chamadas, possíveis strings que são hardcoded na sessão .rdata ou até mesmo .data

- Uma vez que já encontrei certos valores de endereço e funcionalidades pelo CheatEngine, é possível rastrear essas funções e variáveis de forma estática também, assim você consegue cruzar referências e ver todos os pontos onde essas funções/variáveis são chamadas e utilizadas.

![ida_pro](/assets/ida.png)


# Conclusão

Não acho que exista uma mentalidade certa ou errada para alcançar um bom nível de engenharia reversa, acredito fortemente que com a prática você começa a desenvolver os seus próprios métodos de pesquisa e de entendimento de como software e hardware funcionam em conjunto e como você pode tirar vantagem disso para conseguir alcançar seus objetivos.

A ideia deste texto é mostrar um pouco da minha mentalidade enquanto explicando o funcionamento do jogo e das ferramentas para o meu querido amigo Tony Linguiça.


Pretendo iniciar lives focadas em engenharia reversa com o Ghost of Tsushima na Twitch, e por lá, estarei super aberto a explicar a fundo como as ferramentas funcionam e como eu penso quando estou analisando jogos.

Abraços!