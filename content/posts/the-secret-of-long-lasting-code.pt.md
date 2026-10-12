+++ 
date = 2023-02-05T03:07:00+01:00
title = "O segredo de um código duradouro"
categories = ["Coding"]
+++

Não posso falar por todos os desenvolvedores de software do mundo, mas imagino que a maioria de nós já teve que lidar com aplicações legadas em algum momento da carreira.

Mas, antes de entrar no assunto, precisamos de uma definição de software legado, e o [Gartner](https://www.gartner.com/en/information-technology/glossary/legacy-application-or-system) diz:

> Um sistema de informação que pode ser baseado em tecnologias ultrapassadas, mas que é crítico para as operações do dia a dia. Substituir aplicações e sistemas legados por sistemas baseados em tecnologias novas e diferentes é um dos desafios mais significativos dos profissionais de sistemas de informação (SI). À medida que as empresas atualizam ou trocam suas tecnologias, elas precisam garantir a compatibilidade com sistemas antigos e formatos de dados que ainda estão em uso.

Agora, vamos revisar essa definição para entender as premissas e os motivos para um software ser considerado legado.

> Software baseado em tecnologias ultrapassadas, mas crítico para as operações do dia a dia.

Para mim, isso é meio que um paradoxo. O software não deveria usar tecnologias ultrapassadas se é crítico para as operações do dia a dia. Então, ou tanto as operações do dia a dia quanto a aplicação são antigas, e aí a coisa toda é antiga e precisa ser revista, ou o software levou tempo demais para ser desenvolvido e já não atende mais aos requisitos.

No fim das contas, tudo se resume a com que frequência você revisa os requisitos do software. Continue questionando se a sua forma de trabalhar ainda é a correta, ou se a tecnologia ainda é adequada para o seu problema. Ninguém deveria esperar que uma solução específica dure para sempre.

> Substituir aplicações e sistemas legados por sistemas baseados em tecnologias novas e diferentes.

Do meu ponto de vista, substituir um sistema representa o fim da linha. Significa que alguém tomou a péssima decisão de não investir tempo e dinheiro para manter o processo bem ajustado. Ou que alguém fez uma escolha técnica/arquitetural errada. Ou que alguém analisou mal os requisitos. No fim, já é tarde demais para corrigir isso.

> À medida que as empresas atualizam ou trocam suas tecnologias, elas precisam garantir a compatibilidade com sistemas antigos e formatos de dados que ainda estão em uso.

Bom, isso não é nenhuma surpresa e não é um problema exclusivo de aplicações legadas. Não importa o tamanho da aplicação ou a tecnologia que você usa, integração é sempre um desafio. É muito mais uma questão de coordenação e exige atenção constante. E pode quebrar a qualquer minuto, nos lugares mais inesperados.

Resumindo, **decisões ruins são a única causa raiz de um sistema legado**. O software não é o legado. O legado (ou o fardo) é a mentalidade que manteve o software do jeito que estava até ele virar legado.

## Refatoração / Atualização contínua

### Do ponto de vista do negócio

Os requisitos evoluem e exigem revisões regulares, muitas vezes precisando de refatoração para alinhar o código a essas mudanças. Isso pode sobrecarregar o analista de negócios, já que a nova funcionalidade precisa ser compatível com as existentes. E a refatoração também pode aumentar o risco de introduzir defeitos.

Por outro lado, quando escolhemos o caminho mais curto para evitar melhorias necessárias, podemos acabar adiando correções de bugs existentes ou aumentando a dívida técnica, o que também é perigoso para o projeto.

Então, para os stakeholders do negócio, refatoração é um investimento. Histórias bem escritas e cenários de teste bem definidos vão guiar os desenvolvedores na direção certa, e testes BDD e E2E são ótimas ferramentas para detectar regressões. Então, não tenha medo do novo.

### Do ponto de vista do desenvolvimento

Escrever código é desafiador. Melhorar um código existente pode ser duas vezes mais desafiador, dependendo do quão distante você está dos novos requisitos. Mexer no código de outra pessoa pode ser até perigoso, já que talvez você não tenha essa pessoa por perto para te orientar. Tudo isso deixa qualquer desenvolvedor ansioso.

Programadores comuns vão ficar na zona de conforto e deixar tudo como está. Engenheiros excepcionais vão encarar a tarefa, mas podem falhar na primeira tentativa. No entanto, com perseverança suficiente, eles vão se destacar.

Para mim, refatorar é repensar e aprender. Isso mantém todos os desenvolvedores, seniores e juniores, atualizados sobre a tecnologia e o negócio. Quanto mais você refatora, mais simples e rápido fica desenvolver novas funcionalidades.

### Riscos de não melhorar continuamente o seu código

Bom, a sua dívida técnica aumenta. E dívida técnica é um conceito que se refere ao custo de manter um sistema de software por causa de soluções rápidas e ineficientes para os problemas. Ela representa o equilíbrio entre velocidade de desenvolvimento a curto prazo e excelência técnica a longo prazo. Para mais detalhes, confira estes links:
- https://martinfowler.com/bliki/TechnicalDebt.html
- https://www.atlassian.com/agile/software-development/technical-debt

### Benefícios de fazer isso

Refatorar/atualizar não é só reescrever código ou trocar uma tecnologia por outra. É uma questão de estabilidade. Atacar a dívida técnica fortalece a manutenibilidade, a thread safety, a segurança, a escalabilidade e o baixo consumo de recursos. Todos esses são fatores cruciais para um projeto duradouro.

Mas, além disso, manter um projeto estável no ar traz satisfação, visibilidade e reconhecimento. Nada é mais gratificante do que um trabalho bem feito. E você e o seu projeto vão se tornar referência para colegas e stakeholders.

### O que fazer, então

Cada projeto é único, mas existem alguns passos a seguir:
- Avalie a sua situação e prepare um plano. Você quer saber onde está e o que fazer antes de começar a jornada;
- Quebre tudo em pedaços pequenos. Vai ser mais fácil de ser aceito e validado. Você pode começar por componentes isolados ou corrigindo código duplicado. O [Clean Code](https://www.oreilly.com/library/view/clean-code-a/9780136083238/) traz vários insights;
- Garanta uma boa cobertura de testes, ela vai te ajudar a evitar regressões. Se a cobertura de testes não for suficiente, [TDD (Test-driven Development)](https://www.agilealliance.org/glossary/tdd) é uma boa prática para começar a aplicar aqui;
- Melhore o monitoramento e a rastreabilidade da sua aplicação. Não só para se preparar para possíveis novos bugs, mas também para enxergar a sua situação atual. Para monitoramento, sugiro o [Prometheus](https://prometheus.io/). Para rastreabilidade no Spring Boot, o [Sleuth](https://spring.io/projects/spring-cloud-sleuth) é a escolha mais fácil. Leia este [artigo](https://medium.com/javarevisited/distributed-tracing-in-microservices-spring-boot-125272b58ad8) para entender o conceito de Distributed Tracing.

Fora isso, não há muito o que fazer além de fazer o seu trabalho como ele deve ser feito. Bons princípios de programação, clean code, testes significativos, code review, pair programming, Sonar, IDEs inteligentes, IA... Use todas as armas que tiver para garantir que o código que você escreve esteja bem escrito e bem coberto.

## Conclusão

A regra de ouro aqui é: nunca pare de experimentar coisas novas, ou você ficará preso no passado. E, muito provavelmente, todo mundo vai desperdiçar muito tempo e dinheiro fazendo a mesma coisa que você fez, de novo e de novo. Em algum momento, o seu software vai se tornar um sistema legado e todo mundo vai reclamar. Eventualmente, ele será substituído.

Então, por favor, não tenha medo de mudar o código. Nenhum trecho de código é genérico o suficiente para cobrir todas as possibilidades, nem tão bem escrito a ponto de ser a melhor solução para os requisitos.

**See you | Bis geschwënn | Até mais | À bientôt**
