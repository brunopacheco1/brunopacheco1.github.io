+++
date = 2019-11-18T22:05:30+01:00
title = "Quarkus, Kotlin e GraalVM"
description = "Criando um jogo Snake multiplayer para testar Quarkus, Kotlin e imagens nativas do GraalVM, e o que isso me mostrou sobre rodar Java na nuvem."
categories = ["Coding"]
+++

Durante uma conversa com o [László](https://github.com/nerg4l), pedi a ele algumas ideias para praticar Quarkus e GraalVM, e ele sugeriu programar o clássico jogo Snake (ou Worms, como eu chamo :P).

### Motivações

Por que [Quarkus](https://quarkus.io)? Li um artigo dizendo que o Quarkus é uma solução ótima e promissora para Java na nuvem, já que reduz drasticamente o footprint da aplicação e tem uma inicialização absurdamente rápida quando roda em código nativo.

O [GraalVM](https://www.graalvm.org/) é uma VM de uso geral para rodar aplicações escritas em várias linguagens diferentes, permitindo código poliglota. À primeira vista, achei que era algo incrível para testar, e foi o que fiz. No fim, essa primeira tentativa não deu frutos, por causa de um bug estranho em uma aplicação node que eu estava programando na época, então simplesmente desisti naquele momento.

O [Kotlin](https://kotlinlang.org/) não tem nenhum argumento razoável para estar aqui, exceto que é um dialeto Java novinho que eu já estava aprendendo, mas que nunca tinha usado em algo substancial. Então, por que não?

### Começando a aventura

Comecei o projeto apenas lendo os guias do Quarkus.io, que são bem práticos, mas superficiais. O Quarkus é simples de usar, como qualquer framework recente de desenvolvimento web. Tão simples que em cinco minutos eu tinha um serviço REST rodando localmente... claro, uma chamada HTTP que retorna a string "Hello World" não merece ser chamada de serviço, não é?

Para começar seu projeto do zero, eles implementaram um comando de bootstrap. Veja o comando a seguir:

```
mvn io.quarkus:quarkus-maven-plugin:1.0.0.CR1:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=getting-started \
    -DclassName="org.acme.quickstart.GreetingResource" \
    -Dpath="/hello"
```

E uma forma fácil de adicionar novas dependências:

```
mvn quarkus:add-extension -Dextensions="vertx"
```

Seguindo em frente, meus primeiros passos foram: definir o domínio, JPA, H2, alguns endpoints CRUD e alguns DTOs. Depois de ter esse código quadradinho, decidi migrá-lo para Kotlin, e os bugs começaram a aparecer, como esperado.

O primeiro problema que tive que enfrentar foi o JPA. Ele exige classes abertas, e isso não é o padrão no Kotlin. Para resolver, existe uma flag que habilita o suporte a JPA durante a compilação. Não gosto desse tipo de solução, mas a culpa é um pouco minha, já que eu quis usar uma biblioteca Java madura em vez de algo mais no estilo Kotlin.

O segundo problema foi a serialização do payload JSON. Tive alguns problemas com o JSON-B (a opção padrão) e então decidi testar o Jackson. De novo, só por simplicidade, já que eu estava testando um monte de coisas novas.

Funcionou bem e os DTOs ficaram mais limpos do que antes. Porém, foi preciso adicionar outro plugin ao compilador Kotlin (a opção do plugin no-arg) e uma anotação customizada para ajudar o compilador a identificar os DTOs.

```
@Dto
data class MatchMapPlayer(
        var playerId: Long,
        var status: MatchPlayerStatus = MatchPlayerStatus.PLAYING,
        var wormLength: Int = 0,
        var direction: Direction = Direction.DOWN,
        var position: List<MapPoint> = arrayListOf()
)
```

O terceiro problema foi parecido com o do JPA. Todos os managed beans precisam ser abertos. Consegue adivinhar a solução? Sim, mais uma flag do compilador Kotlin.

No fim, a configuração do meu compilador Kotlin ficou assim:

```
<configuration>
  <compilerPlugins>
    <plugin>jpa</plugin>
    <plugin>no-arg</plugin>
    <plugin>all-open</plugin>
  </compilerPlugins>
  <pluginOptions>
    <option>all-open:annotation=javax.ws.rs.Path</option>
    <option>all-open:annotation=javax.enterprise.context.ApplicationScoped</option>
    <option>all-open:annotation=javax.enterprise.context.RequestScoped</option>
    <option>all-open:annotation=javax.ws.rs.ext.Provider</option>
    <option>all-open:annotation=javax.persistence.MappedSuperclass</option>
    <option>all-open:annotation=javax.persistence.Entity</option>
    <option>all-open:annotation=javax.persistence.Embeddable</option>
    <option>no-arg:annotation=com.dev.bruno.worms.dto.Dto</option>
  </pluginOptions>
</configuration>
```

Também criei uma classe abstrata Repository para ter um comportamento comum em todos os repositórios de entidades (copiando o Spring), mas tive um problema com a injeção de dependência na superclasse.

Tentei usar injeção de dependência pelo construtor, passando da classe filha para a superclasse com super, mas não funcionou direito. A solução alternativa foi injetar o `EntityManager` diretamente no campo, assim:

```
    @PersistenceContext
    protected lateinit var em: EntityManager
```

Depois, tive que trocar todos os `@OnToMany` de `List`/`Set` para `MutableList`/`MutableSet`.

E, por fim, não lembro por quê, mas agora todos os campos das classes de domínio e de DTO eram `var` em vez de `val`. Preciso verificar isso de novo.

Também implementei um pouco de código de negócio, que não é importante discutir aqui (mas, se tiver interesse, veja
[este link](https://github.com/brunopacheco1/worms/tree/master/src/main/kotlin/com/dev/bruno/worms/evaluation)).

### E o GraalVM entra em cena...

Depois de passar pela primeira barreira, eu estava pronto para o próximo chefão (o último?): compilar o projeto para código nativo.

Segundo o Quarkus.io, isso não deveria ser complicado, já que eles buscam simplificar a compilação para código nativo com o próprio plugin Maven. Mesmo com guias simplistas, eles não estão mentindo. Mas poderiam ter mencionado a limitação de Reflection no GraalVM, que vou explicar nas próximas linhas.

Para fazer o build da aplicação, você precisa rodar um comando mvn parecido com este:

```
mvn clean package -Pnative -Dnative-image.docker-build=true
```

Aliás, eu estava gerando código nativo em contêiner, por isso esse argumento docker-build. Eles também fornecem um exemplo de Dockerfile.

Achei que nessa etapa isso bastaria, mas não. Depois de rodar o build e a aplicação no Docker, dei de cara com exceções no Jackson. Reflection não é suportado por padrão em código nativo, e levei um tempo para entender por quê, e mais um pouco para encontrar uma forma de resolver.

Aliás, o build consome muita memória e CPU, quase matando meu velho Lenovo. :(

O GraalVM permite usar reflection, mas para isso você precisa adicionar um arquivo de configuração, localizado em `META-INF/native-image/reflect-config.json` (ele pode ficar em outra pasta e com outro nome, mas esse é o caminho padrão).

Primeiro, decidi não escrever esse arquivo à mão e, depois de pesquisar no Google, encontrei este [blog](https://link.medium.com/n2us7AWxF1) apresentando um agente que faz esse trabalho. A ideia é observar a JVM em tempo de execução e gerar a configuração.

Para isso, basta rodar a aplicação com o seguinte comando:

```
$JAVA_HOME/bin/java -agentlib:native-image-agent=config-output-dir=META-INF/native-image your.jar
```

Depois de rodar a aplicação com esse agente, o arquivo gerado ficou extremamente poluído e nem resolveu o problema, já que o agente não adicionou todas as classes necessárias. Então acabei escrevendo o arquivo à mão. E, para minha surpresa, só precisei adicionar as classes de DTO.

O arquivo deve ficar mais ou menos assim:

```
[
  {
    "name": "com.dev.bruno.worms.dto.ExceptionResponse",
    "allDeclaredConstructors": true,
    "allPublicConstructors": true,
    "allDeclaredMethods": true,
    "allPublicMethods": true,
    "allDeclaredClasses": true,
    "allPublicClasses": true,
    "fields": [
      {
        "name": "message",
        "allowWrite": true
      }
    ]
  }
  ...
]
```

### Conclusões

Reconheço os benefícios de usar Kotlin, como nulabilidade, imutabilidade, código menos verboso e um monte de funções novas e amigáveis para quem programa. Porém, eles não são tão importantes quando você tem código limpo e testes eficientes.

Além disso, não gosto de arquivos POM poluídos, e me incomodou ter que adicionar plugins de build, flags de compilador e configurações. Mas me senti culpado por ter usado JPA e Jackson. Talvez, se eu tivesse usado algo mais orientado a Kotlin, não teria tido esses problemas. Vamos ver o que acontece no próximo projeto.

O Quarkus tem mesmo uma inicialização absurdamente rápida, e o consumo de memória é ótimo. Usei uma instância micro no Google Cloud para testar o jogo, e ela aguentou bem a carga, sem derrubar nada. O Quarkus ainda é um pouco imaturo, já que exigiu algumas refatorações quando decidi migrar para versões mais novas. Mesmo assim, é promissor, pois tem um suporte enorme a diferentes tecnologias e bibliotecas. Vou usá-lo de novo.

Gostei do GraalVM à primeira vista, já que o desejo de ter código poliglota está na minha cabeça há um tempo, e agora isso está virando realidade. No fim, não usei esse recurso, mas a geração de código nativo também me fascinou. Só não gostei da forma como ele consome recursos. Compilar código de qualquer outra linguagem para nativo não consome nem 10% do que o GraalVM consumiu da minha máquina (sem julgamentos...). Eu esperaria melhorias nessa área antes de usá-lo em soluções comerciais.

Resumindo: gostei de brincar com todas essas tecnologias novas juntas. Por falta de tempo e disponibilidade, não consegui investigar mais. Talvez melhorar e refatorar o código. A solução final não é ruim e me provou que Java definitivamente pode ser usado na nuvem, é só uma questão de investir em soluções novas, como Quarkus e GraalVM.

Veja o [repositório](https://github.com/brunopacheco1/worms) do projeto para ter uma ideia melhor do que programei. E fique à vontade para comentar por lá.

**See you | Bis geschwënn | Até mais | À bientôt**
