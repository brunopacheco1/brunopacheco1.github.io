+++ 
date = 2023-01-04T00:34:00+01:00
title = "A Covid me levou à minha primeira contribuição open-source"
categories = ["Coding"]
+++

Faz um tempo que não compartilho nada por aqui. Principalmente porque meus interesses e prioridades mudaram. Mas agora estou de volta e espero produzir mais conteúdo.

Vamos voltar a 2020, quando parei de escrever. A era da Covid, baby!

Tinha gente enlouquecendo porque não podia ver outras pessoas. Tinha gente fazendo pão e cerveja em casa. E eu estava me tornando pai pela primeira vez, ou pelo menos começando essa jornada.

Sou engenheiro por amor e desenvolvedor até o osso. Como todo mundo, eu estava procurando algo em que focar meus pensamentos. E um artigo sobre um sistema em COBOL que estava sobrecarregado nos EUA chamou a minha atenção. Claro, eu já tinha ouvido falar de COBOL na faculdade, mas a linguagem mais "hardcore" que eu tinha tocado de leve era C/C++.

Eu estava entediado e não queria ver mais REST, mais linguagens no estilo C, nem mais problemas de arquitetura. Eu queria algo mais básico, fácil de começar e fácil de parar. Por que não COBOL? Pensei que poderia ser útil no futuro.

Então, como qualquer desenvolvedor faria, comecei um curso online. Se você tiver interesse, pode conferir no [LinkedIn Learning](https://www.linkedin.com/learning/learning-cobol-2015).

O que eu descobri? COBOL é uma linguagem elitista e segregadora. Você precisa ter acesso a mainframes para programar e precisa ter acesso a ferramentas pagas para programar e debugar. Existe apenas um compilador open-source para COBOL, o [GnuCOBOL](https://gnucobol.sourceforge.io/). E, quando comecei essa jornada, não havia nenhum debugger open-source.

Fiquei totalmente decepcionado. Como podia ser assim? COBOL é uma linguagem velha pra c****. Então comecei a fuçar em fóruns e encontrei uma plataforma de aprendizado da IBM voltada para COBOL. Se me lembro bem, os alunos tinham acesso a mainframes. Mas uma coisa chamou a minha atenção: havia outra pessoa frustrada reclamando da falta de ferramentas open-source para programar em um computador comum. Ela contou que estava escrevendo um [debugger de COBOL](https://marketplace.visualstudio.com/items?itemName=OlegKunitsyn.gnucobol-debug) para o VSCode.

Testei essa ferramenta e achei a ideia ótima. Mas logo percebi que ela não estava madura o suficiente para debugar nem mesmo uma das tarefas mais simples que fiz no curso online que mencionei.

Bom, como dizemos no Rio, eu estava com a faca e o queijo na mão. Sou desenvolvedor e queria aprender COBOL. Então por que não contribuir com essa ferramenta? Aquele comentariozinho levou meu curso para uma direção completamente diferente. Logo esqueci o "como ler um arquivo em COBOL" e comecei a me perguntar "como o GnuCOBOL transpila código para C", "como debugar C na linha de comando" e "como o gdb se integra com o VSCode".

No meio do projeto, eu já estava em contato com os desenvolvedores do GnuCOBOL, pedindo novas funcionalidades no compilador para ajudar a debugar os tipos de dados float e binários suportados por ele. Também estava adicionando suporte a expressões JavaScript na view de inspeção do debugger. E, nas minhas contribuições mais recentes, eu estava trabalhando no debug de processos remotos e em túneis SSH.

Tudo isso, todas aquelas longas noites programando, só por causa de um comentário em um fórum sobre COBOL.

E sabe de uma coisa? Foi uma experiência incrível. Aprendi muita coisa, talvez nada diretamente útil para o meu dia a dia atual, mas pelo menos contribuí com um projeto open-source, impactando a vida de milhares de outros desenvolvedores anônimos.

**See you | Bis geschwënn | Até mais | À bientôt**
