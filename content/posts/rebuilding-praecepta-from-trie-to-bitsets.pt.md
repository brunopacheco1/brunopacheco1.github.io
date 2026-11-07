+++
date = 2026-11-07T12:00:00+01:00
title = "Reconstruindo o Praecepta: de uma Trie a bitsets, e um benchmark justo contra Camunda e Drools"
description = "O Praecepta 0.1.0 está no Maven Central: um motor de tabelas de decisão DMN reconstruído com bitsets, medido de forma justa contra o Camunda 8 e o Drools, com as suas limitações ditas com clareza. Como usar, quão aberto é o projeto, e o que pode vir a seguir."
categories = ["Coding"]
+++

Em janeiro de 2023 escrevi sobre [trocar motores DMN por uma Trie](/pt/posts/optimizing-decision-making-with-trie-tree/). A ideia funcionou, e os números pareciam ótimos. Revendo tudo este ano, encontrei dois problemas. A memória da Trie explodia conforme as tabelas cresciam: uma tabela gerada com 1.000 regras ocupava **mais de 1 GB**. E o benchmark não era justo: o Praecepta rodava com uma hit policy diferente da do Camunda e do Kogito, e dois dos serviços liam uma das entradas do campo errado.

Então eu o reconstruí. O [Praecepta 0.1.0](https://github.com/brunopacheco1/praecepta) já está no Maven Central: um motor Java pequeno, sem dependências, para tabelas de decisão DMN. Este post cobre o que mudou, um benchmark contra o Camunda e o Drools que me esforcei para deixar justo (inclusive onde ele favorece o Praecepta), o que esses motores fazem e o Praecepta não, quão aberto é o projeto, e o que pode vir a seguir. Nessa última parte, eu adoraria ouvir a sua opinião.

## De uma Trie a bitsets

Uma tabela de decisão é uma lista de regras. Cada regra testa algumas colunas de entrada (`country é "PT" ou "ES"`, `age em [18..65)`, ou "qualquer valor") e produz saídas. A Trie guardava todos os caminhos possíveis pelas colunas, e cada célula "qualquer valor" multiplicava os caminhos. Essa multiplicação era o problema de memória, e nenhum ajuste fino resolve isso.

O novo núcleo usa **bitsets**. Cada regra ganha um bit. Para cada coluna, cada valor que as regras conseguem distinguir ganha uma máscara: o conjunto de regras que aceitam aquele valor.

| Coluna `country` | regra 1 | regra 2 | regra 3 | regra 4 |
|---|:---:|:---:|:---:|:---:|
| `"PT"` | 1 | 0 | 1 | 1 |
| `"FR"` | 0 | 1 | 1 | 0 |
| qualquer outro valor | 0 | 0 | 1 | 1 |

A regra 1 testa `"PT"`, a regra 2 testa `"FR"`, a regra 3 é "qualquer valor" (por isso está em todas as máscaras) e a regra 4 é `not("FR")`. Avaliar uma entrada é buscar uma máscara por coluna e fazer um AND entre elas. Um AND de 64 bits verifica 64 regras de uma vez, e a tabela de 119 regras que vem com o projeto cabe em duas palavras. A memória cresce com colunas × valores × regras, nunca com o produto dos valores, e a tabela de 1.000 regras que precisava de 1 GB agora ocupa **0,4 MB** e é montada em cerca de 80 ms.

A hit policy também sai direto dos bits. Os bits seguem a ordem das regras (ou a ordem de prioridade), então FIRST é "o bit ligado mais baixo", uma única instrução da CPU.

Números e datas funcionam do mesmo jeito: os limites que as regras mencionam (`18`, `65`, `date("2025-01-01")`) dividem cada coluna em intervalos, uma busca binária encontra o intervalo, e cada intervalo tem a sua máscara.

## O que mais mudou

- **Hit policies estritas do DMN 1.3:** UNIQUE, ANY, PRIORITY, FIRST, OUTPUT ORDER, RULE ORDER e COLLECT com SUM, MIN e MAX.
- **Células em FEEL** em colunas de texto, número e data: listas, `not(...)`, comparações, intervalos e `null`.
- **Validação na montagem da tabela:** regras sobrepostas, saídas conflitantes, regras que nunca disparam e *lacunas* (entradas que nenhuma regra cobre, com um exemplo).
- **Importação e exportação de DMN:** o Praecepta lê arquivos DMN 1.1–1.5, inclusive os do Camunda Modeler.
- **Um plugin Maven** que valida as tabelas no build e gera records tipados de entrada e saída, para que a sua aplicação leve só o motor.
- **Java 21+**, com o motor como módulo JPMS. Tudo é testado no Java 21, 25 e 27.

## O benchmark

### Como foi medido
- **As mesmas regras para todos os motores.** Uma única lista de regras é escrita em DMN para o Camunda e o Drools. Antes de qualquer medição, todos os motores avaliam a carga inteira e os resultados são comparados; o benchmark é abortado se algum motor discordar.
- **Configurações padrão e suportadas:** o motor DMN do Camunda 8 (dmn-scala 1.12.6, o mesmo que o Zeebe usa) e o runtime DMN do Drools 10.2.0. Cada motor lê o seu modelo uma única vez; só a avaliação é medida.
- **A mesma entrada para todos:** um `Map` de valores. O caminho mais rápido do Praecepta (classes geradas com enums) aparece separado, identificado como o seu melhor caso.
- **Cinco tabelas:** a tabela real de 119 regras de 2023 (com valores ofuscados), tabelas geradas só de texto com 100 e 1.000 regras, e tabelas geradas de 100 e 1.000 regras que misturam colunas de texto, número e data com comparações, intervalos e `not(...)`.
- **Código publicado:** o Praecepta 0.1.0 direto do Maven Central.
- **Máquina:** Apple M2 Pro, 16 GB, macOS 26.7, OpenJDK 27. JMH: aquecimento de 5 × 2 s e medição de 5 × 3 s, 2 forks, uma thread.

### Latência

{{< chart >}}
type: 'bar',
data: {
  labels: ['real (119 regras)', 'synthetic-100', 'synthetic-1000', 'mixed-100', 'mixed-1000'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.188, 0.150, 0.226, 0.485, 0.684], backgroundColor: '#0e7490' },
    { label: 'Camunda 8 (dmn-scala 1.12.6)', data: [16.70, 19.47, 127.0, 37.21, 282.8], backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [59.79, 16.38, 143.9, 32.38, 276.1], backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Tempo por avaliação (µs, escala logarítmica, menor é melhor)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'µs' } } }
}
{{< /chart >}}

| µs por avaliação (± erro de 99,9%) | real | synthetic-100 | synthetic-1000 | mixed-100 | mixed-1000 |
|---|---:|---:|---:|---:|---:|
| **Praecepta 0.1.0** | **0,19** ± 0,01 | **0,15** ± 0,01 | **0,23** ± 0,01 | **0,49** ± 0,02 | **0,68** ± 0,04 |
| Camunda 8 | 16,7 ± 0,1 | 19,5 ± 0,2 | 127 ± 3 | 37,2 ± 2,2 | 283 ± 25 |
| Drools 10.2.0 | 59,8 ± 6,1 | 16,4 ± 2,7 | 144 ± 42 | 32,4 ± 1,2 | 276 ± 17 |

O Praecepta é entre **70×** (mixed-100 contra o Drools) e **640×** (synthetic-1000 contra o Drools) mais rápido. Repare em duas coisas. Primeiro, o tempo do Praecepta quase não cresce de 100 para 1.000 regras, enquanto o dos outros cresce mais ou menos linearmente. Segundo, colunas de número e data custam ao Praecepta 2–3× mais que colunas de texto, porque uma busca binária em decimais é mais lenta que uma busca em hash; para isso já existe [uma issue](https://github.com/brunopacheco1/praecepta/issues/16).

O Drools foi o motor mais instável (±42 µs no synthetic-1000). Para verificar que o resultado não é um artefato da minha máquina, rodei a mesma suíte num runner do GitHub Actions (x86, Java 25). Os números absolutos mudam, mas a ordem se mantém: Praecepta 0,15–0,65 µs; Camunda 13,7–274 µs; Drools 12,4–218 µs.

### Memória e tempo de montagem

{{< chart >}}
type: 'bar',
data: {
  labels: ['real (119 regras)', 'synthetic-100', 'synthetic-1000', 'mixed-100', 'mixed-1000'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.2, 0.3, 0.4, 0.4, 0.9], backgroundColor: '#0e7490' },
    { label: 'Camunda 8', data: [4.6, 4.9, 16.1, 5.0, 16.5], backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [6.6, 7.0, 13.8, 7.5, 15.8], backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Heap retido após a montagem (MB, menor é melhor)' } },
  scales: { y: { title: { display: true, text: 'MB' } } }
}
{{< /chart >}}

| Tempo de montagem | real | synthetic-100 | synthetic-1000 | mixed-100 | mixed-1000 |
|---|---:|---:|---:|---:|---:|
| **Praecepta 0.1.0** | **39 ms** | **60 ms** | **82 ms** | **68 ms** | **167 ms** |
| Camunda 8 | 437 ms | 488 ms | 700 ms | 534 ms | 808 ms |
| Drools 10.2.0 | 336 ms | 401 ms | 628 ms | 491 ms | 1.060 ms |

Cada motor é montado numa JVM nova, e "heap retido" é o heap ainda em uso depois de montá-lo e rodar uma coleta de lixo.

### O melhor caso do Praecepta

Usar um `Map` custa duas buscas em hash por coluna: uma para ler o valor e outra para achar a sua máscara. Com o plugin Maven, as entradas são records gerados, e colunas de texto com uma lista de valores viram enums, cujas máscaras são encontradas pela posição em vez de por hash:

{{< chart >}}
type: 'bar',
data: {
  labels: ['Map de textos', 'record gerado (textos)', 'record gerado (enums)'],
  datasets: [{ label: 'ns por avaliação, tabela real', data: [231, 142, 101], backgroundColor: ['#94a3b8', '#38bdf8', '#0e7490'] }]
},
options: {
  plugins: { title: { display: true, text: 'Formas de entrada do Praecepta na tabela real (ns, menor é melhor)' }, legend: { display: false } }
}
{{< /chart >}}

Isso dá cerca de **100 ns** por decisão na tabela real. Para manter a comparação justa, a comparação entre motores acima usa entradas `Map` para todos, inclusive o Praecepta.

## Sendo justo: o que os números não dizem

**Onde a comparação favorece o Praecepta:**
- **O Praecepta faz menos trabalho.** O Camunda e o Drools são interpretadores FEEL de uso geral. Mesmo quando uma célula é um teste simples, eles a avaliam com uma maquinaria capaz de lidar com expressões arbitrárias, e devolvem resultados mais ricos. O Praecepta só suporta o que consegue indexar.
- **O teste é restrito.** Ele roda dentro do processo, numa única thread, com a JVM já aquecida. Um serviço real soma HTTP e serialização, que normalmente custam bem mais que 20 µs, então em muitos sistemas o motor nem é o gargalo.
- **Nem toda opção foi medida.** O modelo compilado (executável) do Drools, gerado no build pelo Kogito, não foi medido; ele pode ser mais rápido que o caminho de DMN em runtime que usei. A opção experimental "alpha network" do Drools deu respostas erradas nessas tabelas na versão 10.2.0, então o benchmark se recusou a medi-la.

**O que o Camunda e o Drools fazem e o Praecepta não:**
- **FEEL completo:** funções, aritmética, contextos, listas e testes envolvendo várias colunas (`income > 2 * rent`). As células do Praecepta são *testes unários simples* do FEEL, sobre uma única coluna.
- **Grafos de requisitos de decisão:** decisões que alimentam outras decisões, modelos de conhecimento de negócio e expressões em caixa.
- **Mais tipos:** colunas booleanas, de hora e de data-hora (planejadas).
- **Ferramentas e serviços:** modeladores visuais, serviços de decisão, motores de processo, auditoria e explicação das decisões.
- **Conformidade comprovada:** o Drools roda o kit oficial de conformidade do DMN; o Praecepta ainda não foi testado com ele.
- **Maturidade:** comunidades grandes, anos de uso em produção e suporte comercial.

Se as suas decisões são principalmente tabelas de decisão com testes simples nas células, e latência, memória ou tempo de inicialização importam, vale a pena olhar o Praecepta. Se você precisa do DMN completo, use o Camunda ou o Drools.

## Como usar

Adicione o motor (ele não tem dependências):

```xml
<dependency>
  <groupId>io.github.brunopacheco1</groupId>
  <artifactId>praecepta-engine</artifactId>
  <version>0.1.0</version>
</dependency>
```

Carregue um arquivo DMN, associe as entradas e avalie:

```java
record Customer(String country, Integer age) {}

DecisionTable table;
try (var dmn = Files.newInputStream(Path.of("discount.dmn"))) {
    table = DmnReader.read(dmn);
}
var engine = PraeceptaEngine.<Customer, String>builder()
        .table(table)                              // hit policy, tipos das colunas, listas de valores, regras
        .input("country", Customer::country)
        .numberInput("age", Customer::age)
        .output(output -> output.values().get("tier"))
        .build();

engine.evaluate(new Customer("PT", 30));          // por exemplo, [gold]
engine.validationReport();                         // sobreposições, regras inalcançáveis, lacunas
```

Ou deixe o plugin Maven compilar as suas tabelas (Excel, CSV ou DMN) no build:

```xml
<plugin>
  <groupId>io.github.brunopacheco1</groupId>
  <artifactId>praecepta-maven-plugin</artifactId>
  <version>0.1.0</version>
  <executions><execution><goals><goal>generate</goal></goals></execution></executions>
  <configuration>
    <packageName>com.acme.pricing</packageName>
    <tables>
      <table><source>src/main/rules/discount.dmn</source></table>
    </tables>
  </configuration>
</plugin>
```

```java
Optional<DiscountOutput> discount = Discount.evaluate(new DiscountInput(DiscountInput.Country.PT, new BigDecimal(30)));
```

O build falha se a tabela tiver erros (por exemplo, duas regras UNIQUE que se sobrepõem), e só o motor é necessário em runtime. O [README](https://github.com/brunopacheco1/praecepta#readme) tem os detalhes.

## Quão aberto é o projeto?

Eu queria um projeto pequeno e fácil de confiar. Na prática, isso significa:

- **Licença:** Apache-2.0. O repositório segue o [REUSE](https://reuse.software): cada arquivo diz quem o escreveu e sob qual licença.
- **Tudo em público:** o código, a CI, o [plano de modernização](https://github.com/brunopacheco1/praecepta/blob/main/docs/MODERNISATION.md) fase a fase, o benchmark e as issues.
- **Integridade das versões:** cada artefato é assinado com GPG e vem com um SBOM (CycloneDX), proveniência de build (SLSA, verificável com `gh attestation verify`) e avisos de terceiros. Os builds são reproduzíveis.
- **Verificações em cada mudança:** testes em três versões do Java, testes do plugin em três versões do Maven, um quality gate do SonarCloud, CodeQL, verificação de licenças e de vulnerabilidades das dependências, e fuzzing dos dois parsers que leem entradas não confiáveis.
- **[OpenSSF Scorecard](https://scorecard.dev/viewer/?uri=github.com/brunopacheco1/praecepta): 7,8 / 10.**

  ![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/brunopacheco1/praecepta/badge)

O item mais baixo do Scorecard é **code review**, e vale explicar. Qualquer pessoa pode fazer um fork do projeto e abrir um pull request. O que essa verificação mede é outra coisa: se cada mudança que chegou ao `main` foi aprovada por alguém além do autor. Hoje isso é zero, porque sou o único mantenedor e ninguém pode aprovar as próprias mudanças. Cada contribuição que eu revisar aumenta esse número, e um segundo mantenedor o completaria. O que me leva à próxima parte.

## Quer ajudar?

O [guia de contribuição](https://github.com/brunopacheco1/praecepta/blob/main/CONTRIBUTING.md) explica como compilar e testar. Boas primeiras issues:

- [intervalos de texto nas células](https://github.com/brunopacheco1/praecepta/issues/13);
- [uma agregação COUNT](https://github.com/brunopacheco1/praecepta/issues/15);
- [rodar o kit de conformidade do DMN](https://github.com/brunopacheco1/praecepta/issues/22).

Revisores e um co-mantenedor também são muito bem-vindos.

## O que pode vir a seguir?

Estas são as funcionalidades que acho que seriam úteis. **Me diga quais você usaria**: um 👍 na issue já basta, e um comentário se quiser construí-la.

- [Colunas booleanas, de hora e de data-hora](https://github.com/brunopacheco1/praecepta/issues/12)
- [Intervalos de texto nas células](https://github.com/brunopacheco1/praecepta/issues/13)
- [Grafos de requisitos de decisão (decisões encadeadas)](https://github.com/brunopacheco1/praecepta/issues/14)
- [Uma agregação COUNT](https://github.com/brunopacheco1/praecepta/issues/15)
- [Colunas de número e data mais rápidas](https://github.com/brunopacheco1/praecepta/issues/16)
- [Bitmaps comprimidos para tabelas muito grandes](https://github.com/brunopacheco1/praecepta/issues/17)
- [Um plugin para Gradle](https://github.com/brunopacheco1/praecepta/issues/18)
- [Suporte a native image do GraalVM](https://github.com/brunopacheco1/praecepta/issues/19)
- [Um starter para Spring Boot](https://github.com/brunopacheco1/praecepta/issues/20)
- [Um playground online](https://github.com/brunopacheco1/praecepta/issues/21)
- [Rodar o kit de conformidade do DMN](https://github.com/brunopacheco1/praecepta/issues/22)

## Para encerrar

A Trie de 2023 provou a ideia e bateu num limite. Os bitsets a levaram mais longe: nessas tabelas, de 70 a 640 vezes mais rápido que motores DMN de uso geral, com 18 a 40 vezes menos memória, e honesto sobre o que não faz. Experimente, me conte onde ele quebra, e me diga o que você gostaria que ele fizesse a seguir.

**See you | Bis geschwënn | Até mais | À bientôt**
