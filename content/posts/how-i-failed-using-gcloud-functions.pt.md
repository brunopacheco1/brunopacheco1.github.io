+++ 
date = 2020-02-12T20:37:41+01:00
title = "Como eu falhei ao usar Google Cloud Functions"
+++

Este post é uma continuação do meu artigo anterior sobre [Quarkus, Kotlin e GraalVM](/posts/pt/quarkus-kotlin-and-graalvm).

Fiquei orgulhoso por ter terminado o jogo da Snake, que estava funcionando no modo multiplayer. No entanto, eu queria melhorar a experiência do jogo abordando dois pontos: a interface e a escalabilidade. Decidi focar na escalabilidade, mas infelizmente falhei.

Usei [Google Cloud Functions](https://firebase.google.com/docs/functions), [Firebase](https://firebase.google.com/), [Firestore](https://firebase.google.com/docs/firestore) e [Google Pub/Sub](https://cloud.google.com/pubsub).

## Motivação
No projeto original, usei [VertX](https://vertx.io/) para o PubSub em memória, gerenciando os eventos do jogo. VertX é uma biblioteca poderosa e versátil que me permitiu escrever um código menos acoplado, utilizando recursos de forma eficiente e simplificando o trabalho com EventSource e Server Sent Events, resultando em uma interface verdadeiramente reativa.

No entanto, essa não era uma solução escalável, pois tudo era feito em memória. Existem milhares de maneiras de escalar, mas eu queria testar as Cloud Functions para entender o desempenho e avaliar se o código seria mais fácil de manter.

## O que foi feito?

Programar as funções foi a parte mais rápida, pois a ideia já estava lá graças ao projeto anterior. No entanto, a curva de aprendizado foi maior, pois precisei aprender a usar o Firebase para publicar as funções, o Firestore para persistir as partidas e o Pub/Sub para transmitir os eventos.

O código original está disponível [aqui](https://github.com/brunopacheco1/worms) e o código serverless [aqui](https://github.com/brunopacheco1/worms-serverless).

## O que eu descobri?

Se você for corajoso o suficiente para comparar os códigos, conferira diretamente os repositórios, mas aqui estão minhas descobertas:

Certamente, o código ficou muito menor com as funções lambda. O código em Kotlin/Quarkus era mais verboso porque eu precisei codificar todas as camadas (domínio, DTOs, repositórios, serviços, controladores, tratamento de exceções). Com as funções lambda, precisei apenas das classes de domínio (especificamente para inicializar valores padrão), avaliadores de estado e controladores. Acredito que isso se deve à linguagem - Javascript é mais simples e facilita a manipulação de objetos, e o Firebase me ofereceu maneiras mais fáceis de expor os endpoints e conectar todos os componentes.

Usar o Firebase e outros produtos do Google Cloud foi fácil e econômico, com documentação clara e integração simples. No entanto, o desempenho das Google Cloud Functions ficou abaixo do esperado. Comparando, hospedar o jogo Worms em uma pequena instância proporcionava uma experiência de jogo satisfatória, enquanto as Google Cloud Functions tornaram impossível jogar.

## O que eu deveria ter feito?

- Eu deveria ter adicionado testes de estresse para ter uma forma confiável de comparar ambos os projetos.
- Eu deveria ter identificado o que precisava ser melhorado em ambos os projetos. Minhas considerações iniciais foram que o poder de CPU reduzido e a latência ao conectar-se ao Firestore e ao Pub/Sub problemáticos. As primeiras chamadas mais lentas também poderiam ser devido à alocação de recursos.
- Eu deveria ter documentado adequadamente o benchmark de ambas as soluções para ter evidências concretas a compartilhar.

**See you | Bis geschwënn | Até mais | À bientôt**
