+++
date = 2026-11-07T12:00:00+01:00
title = "Reconstruindo o Praecepta: de uma Trie a bitsets, e um benchmark justo contra Camunda e Drools"
description = "O Praecepta 0.1.0 está no Maven Central: um motor de tabelas de decisão DMN reconstruído com bitsets, medido de forma justa contra o Camunda 8 e o Drools, com as suas limitações ditas com clareza. Como usar, quão aberto é o projeto, e o que pode vir a seguir."
categories = ["Coding"]
+++

Em janeiro de 2023 escrevi sobre [otimizar a tomada de decisão com uma Trie](/pt/posts/optimizing-decision-making-with-trie-tree/). A ideia funcionou em princípio: parte dela foi implementada, e ainda é mantida, na [Pictet Tech](https://www.pictet.com/mc/en/about/pictet-tech). Os números pareciam ótimos, como você pode ver no meu post anterior.

Revendo meus repositórios antigos em busca de oportunidades para testar o [Claude](https://claude.ai/new), decidi explorar o maior problema daquela solução: a memória da Trie explodia conforme as tabelas cresciam; uma tabela gerada com 1.000 regras ocupava **mais de 1 GB**. E, durante o desenvolvimento, o Claude me disse que o primeiro benchmark não era totalmente justo, já que o Praecepta rodava com uma hit policy diferente da do Camunda e do Kogito, e dois dos serviços liam uma das entradas do campo errado.

Então refatorei o motor e o benchmark, e completei o plugin Maven que compila as tabelas no build. O resultado da refatoração está no [Praecepta 0.1.0](https://github.com/brunopacheco1/praecepta), no Maven Central. Os princípios continuam os mesmos: um motor Java pequeno e sem dependências para tabelas de decisão DMN. Agora ele também tem um [site](https://bruno.pacheco.lu/praecepta/) para o guia e a documentação da API.

Este post cobre o que mudou, um benchmark contra o Camunda e o Drools que me esforcei para deixar justo (inclusive onde ele favorece o Praecepta, e onde o Praecepta é mais lento), e o que esses motores fazem e o Praecepta não. Este post também cobre quão aberto o Praecepta é, e o que pode vir a seguir. Nessa última parte, eu adoraria ouvir a sua opinião.

## De uma Trie a bitsets

Uma tabela de decisão é uma simples lista de regras. Cada regra testa algumas colunas de entrada (`country é "PT" ou "ES"`, `age em [18..65)`, ou "qualquer valor") e produz saídas. A solução anterior, uma implementação própria de uma [Trie](https://www.geeksforgeeks.org/dsa/trie-insert-and-search/), guardava todos os caminhos possíveis pelas colunas, e cada célula "qualquer valor" multiplicava os caminhos. Essa multiplicação era o problema de memória!

Tentei algumas coisas na época, inclusive computação quântica, com o [qengine](https://github.com/brunopacheco1/qengine/tree/main/src/main/java/com/github/brunopacheco1/qengine): codificar a entrada como bits e cada regra como portas lógicas quânticas, para ver se um circuito quântico superava a Trie. Ele só rodou num simulador, mas a ideia ficou comigo: casar regras é lógica booleana sobre bits. O índice de bitsets é a versão clássica e prática dessa ideia.

No fim, parei de pensar nisso. Mas agora, em 2026, e com o apoio do Claude, o novo núcleo usa **bitsets**. Cada regra ganha um bit. Para cada coluna, cada valor que as regras conseguem distinguir ganha uma máscara: o conjunto de regras que aceitam aquele valor.

| Coluna `country` | regra 1 | regra 2 | regra 3 | regra 4 |
|---|:---:|:---:|:---:|:---:|
| `"PT"` | 1 | 0 | 1 | 1 |
| `"FR"` | 0 | 1 | 1 | 0 |
| qualquer outro valor | 0 | 0 | 1 | 1 |

A regra 1 testa `"PT"`, a regra 2 testa `"FR"`, a regra 3 é "qualquer valor" (por isso está em todas as máscaras) e a regra 4 é `not("FR")`. Avaliar uma entrada é buscar uma máscara por coluna e fazer um AND entre elas. Um AND de 64 bits verifica 64 regras de uma vez, e a tabela de 119 regras que vem com o projeto cabe em duas palavras. A memória cresce com colunas × valores × regras, nunca com o produto dos valores, e a tabela de 1.000 regras que precisava de 1 GB agora ocupa **0,6 MB**, e lê-la do DMN e montá-la leva cerca de um quarto de segundo numa JVM nova.

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
- **As mesmas regras, o mesmo arquivo, para todos os motores.** Cada tabela é escrita uma única vez como documento DMN, e todos os motores, inclusive o Praecepta, leem esse documento, como faria uma aplicação que carrega um arquivo `.dmn`. Antes de qualquer medição, todos os motores avaliam a carga inteira e as respostas são comparadas com o Praecepta montado direto das regras. O benchmark é abortado se algum motor discordar.
- **Configurações padrão e suportadas:** o motor DMN do Camunda 8 (dmn-scala 1.12.6, o mesmo que o Zeebe usa) e o runtime DMN do Drools 10.2.0.
- **A mesma entrada para todos:** um `Map` de valores. O caminho mais rápido do Praecepta (classes geradas com enums) aparece separado, identificado como o seu melhor caso.
- **Muitas tabelas:**
  - **Real:** a tabela de 119 regras de 2023 (com valores ofuscados, hit policy FIRST).
  - **Geradas, `synthetic`:** só colunas de texto.
  - **Geradas, `mixed`:** colunas de texto, número e data, com comparações, intervalos e `not(...)`.
  - **Tamanhos:** as tabelas geradas vão de 10 a 10.000 regras e de 5 a 20 colunas. As mesmas 1.000 regras também rodam com cada hit policy, e com entradas que não casam com nenhuma regra.
- **Mais do que uma média:** percentis de latência, memória alocada por decisão, vazão com 1, 4 e 8 threads compartilhando um motor, o tempo de montar um motor, e o tempo até a primeira decisão numa JVM nova.
- **O motor da 0.1.0:** o código do motor é o mesmo da versão 0.1.0.
- **Máquina:** Apple M2 Pro (8 núcleos de desempenho), 16 GB, macOS 26.7.1, OpenJDK 27.0.1. JMH: aquecimento de 5 × 2 s e medição de 5 × 3 s, 3 forks.
- **Tudo é público:** [os resultados](https://github.com/brunopacheco1/praecepta/tree/main/praecepta-benchmarks/results/2026-11-07-apple-m2-pro-full) guardam todos os logs, a saída bruta do JMH e o ambiente, e um único script roda a suíte inteira de novo em cerca de 2,5 horas. Para verificar que o resultado não é um artefato da minha máquina, também rodei o perfil curto num [runner do GitHub Actions](https://github.com/brunopacheco1/praecepta/actions/runs/42391587868) (AMD EPYC, 4 vCPUs, Linux, Java 25). Ele usou um único fork, então os números têm mais ruído, e os tempos absolutos são 1,5–3× maiores, mas a ordem se mantém em todas as tabelas. No mixed-1000, por exemplo, o Praecepta levou 1,0 µs, o Camunda 519 µs e o Drools 421 µs.

### Latência conforme as tabelas crescem

{{< chart >}}
type: 'line',
data: {
  labels: ['10 regras', '100 regras', '1.000 regras', '10.000 regras'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.31, 0.44, 0.64, 1.51], borderColor: '#0e7490', backgroundColor: '#0e7490' },
    { label: 'Camunda 8 (dmn-scala 1.12.6)', data: [7.05, 33.2, 263, 3118], borderColor: '#f97316', backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [7.95, 31.3, 256, 3122], borderColor: '#6366f1', backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Tempo por avaliação nas tabelas mixed (µs, escala logarítmica, menor é melhor)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'µs' } } }
}
{{< /chart >}}

| µs por avaliação | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| real (119 regras) | **0,19** | 19,3 | 43,5 |
| synthetic-10 | **0,15** | 4,20 | 5,89 |
| synthetic-100 | **0,16** | 19,1 | 14,5 |
| synthetic-1000 | **0,25** | 146 | 126 |
| synthetic-10000 | **0,67** | 1.333 | 1.419 |
| mixed-10 | **0,31** | 7,05 | 7,95 |
| mixed-100 | **0,44** | 33,2 | 31,3 |
| mixed-1000 | **0,64** | 263 | 256 |
| mixed-10000 | **1,51** | 3.118 | 3.122 |

A diferença depende do tamanho da tabela:
- **Praecepta:** de 10 para 10.000 regras mixed, o tempo vai de 0,31 para 1,51 µs, 5× mais lento para 1.000× mais regras.
- **Camunda e Drools:** o tempo cresce mais ou menos com o número de regras, de cerca de 7 µs para cerca de 3.100 µs.
- **A razão:** o Praecepta é 25–40× mais rápido com 10 regras, 70–120× com 100, 400–600× com 1.000 e cerca de 2.000× com 10.000. Na tabela real, é 104× mais rápido que o Camunda e 233× mais rápido que o Drools.

Mais colunas custam pouco ao Praecepta: com 1.000 regras, passar de 6 para 18 colunas mixed leva o tempo de 0,62 para 1,12 µs. Colunas de número e data custam 2–3× mais que colunas de texto, porque uma busca binária em decimais é mais lenta que uma busca em hash; para isso já existe [uma issue](https://github.com/brunopacheco1/praecepta/issues/16).

51 das 63 medições de latência têm um intervalo de confiança de 99,9% dentro de ±10%. O mais largo é o do Drools no synthetic-10000, com ±33%. Todos estão nos resultados.

### Hit policies, e entradas que não casam com nenhuma regra

| µs por avaliação, 1.000 regras mixed | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| COLLECT | **0,64** | 263 | 256 |
| FIRST | **0,53** | 49,6 | 248 |
| PRIORITY | **0,55** | 287 | 252 |
| RULE ORDER | **0,64** | 285 | 257 |
| UNIQUE (1.000 regras que nunca se sobrepõem) | **0,23** | 161 | 128 |
| Nenhuma regra casa (COLLECT) | **0,32** | 261 | 197 |

- **Praecepta:** a hit policy quase não muda o tempo, porque é uma única operação sobre os bits que casaram. Uma entrada que não casa com nada não custa mais do que uma que casa.
- **Camunda:** é 5× mais rápido com FIRST, o que sugere que ele para na primeira regra que casa.
- **Drools:** leva mais ou menos o mesmo tempo com qualquer hit policy. Quando nada casa, ainda paga cerca de três quartos do tempo de uma entrada que casa, e o Camunda quase o tempo todo.
- **A tabela real** não tem entradas assim: a sua última regra casa com qualquer valor.

### Latência de cauda e memória por decisão

| µs por avaliação | p50 | p99 | p99.9 |
|---|---:|---:|---:|
| **Praecepta**, real | **0,17** | **0,29** | **0,54** |
| Camunda 8, real | 17,7 | 32,4 | 41,1 |
| Drools 10.2.0, real | 51,8 | 69,8 | 85,5 |
| **Praecepta**, mixed-1000 | **0,67** | **1,00** | **2,57** |
| Camunda 8, mixed-1000 | 250 | 323 | 444 |
| Drools 10.2.0, mixed-1000 | 260 | 309 | 386 |

- **Caudas:** o percentil 99,9 do Praecepta no mixed-1000 (2,6 µs) ainda fica cerca de 100× abaixo da mediana dos outros.
- **As decisões mais lentas:** a decisão mais lenta de cada motor levou até alguns milissegundos, muito provavelmente por pausas da JVM ou do sistema operacional, que todos os motores sofrem.
- **Alocação:** em tabelas de até 1.000 regras, o Praecepta aloca de 32 a 466 bytes por decisão. O Camunda aloca de 22 KB a 1,5 MB, e o Drools de 16 KB a 375 KB. Menos lixo significa menos coleta de lixo sob carga.

### Vazão com várias threads

{{< chart >}}
type: 'bar',
data: {
  labels: ['1 thread', '4 threads', '8 threads'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [5374, 21158, 33796], backgroundColor: '#0e7490' },
    { label: 'Camunda 8', data: [61.0, 231, 451], backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [20.9, 78.6, 146], backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Decisões por milissegundo na tabela real, um motor compartilhado por todas as threads (escala logarítmica, maior é melhor)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'decisões / ms' } } }
}
{{< /chart >}}

Os três motores podem ser compartilhados entre threads, e todos escalam.
- **De 1 para 8 threads:** o Praecepta vai de 5.374 para 33.796 decisões por milissegundo (6,3×), o Camunda de 61 para 451 (7,4×) e o Drools de 21 para 146 (7,0×).
- **Locks:** o motor do Praecepta é imutável, então as threads o compartilham sem nenhum.
- **mixed-1000, 8 threads:** 11.721 decisões por milissegundo no Praecepta, contra 29 no Camunda e 23 no Drools.

### Inicialização e montagem

| ms | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| Ler o DMN e montar o motor, JVM aquecida: real | **6,5** | 22,6 | 36,1 |
| Ler o DMN e montar o motor, JVM aquecida: mixed-1000 | **45** | 57 | 437 |
| Primeira decisão numa JVM nova: real | **76** | 469 | 379 |
| Primeira decisão numa JVM nova: mixed-1000 | **240** | 774 | 1.005 |

A "primeira decisão" inicia uma JVM nova, lê o arquivo DMN, monta o motor e avalia uma entrada. Na tabela real, o Praecepta responde em 76 ms, 5–6× antes dos outros. Isso importa para ferramentas de linha de comando, funções serverless e suítes de teste, e bem menos para serviços que ficam rodando por muito tempo.

### Memória

{{< chart >}}
type: 'bar',
data: {
  labels: ['10 regras', '100 regras', '1.000 regras', '10.000 regras'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.6, 0.6, 1.1, 5.0], backgroundColor: '#0e7490' },
    { label: 'Camunda 8', data: [3.6, 4.8, 16.3, 133], backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [6.6, 7.4, 15.9, 108], backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Heap retido após a montagem, tabelas mixed (MB, escala logarítmica, menor é melhor)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'MB' } } }
}
{{< /chart >}}

"Heap retido" é o heap ainda em uso depois de montar o motor numa JVM nova e rodar uma coleta de lixo.
- **Com 10.000 regras:** o Praecepta ocupa 5 MB, contra 133 MB do Camunda e 108 MB do Drools.
- **Em todas as tabelas geradas:** usa de 6 a 88 vezes menos.
- **A tabela real:** ali o Praecepta ocupou pouco demais para a medição conseguir distinguir (menos de 0,5 MB).

### Onde o Praecepta é mais lento: montar tabelas FIRST e PRIORITY

| Tempo de montagem numa JVM nova, 1.000 regras mixed | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| COLLECT | **0,30 s** | 0,87 s | 1,17 s |
| RULE ORDER | **0,27 s** | 0,83 s | 1,07 s |
| FIRST | 3,6 s | **0,85 s** | 1,15 s |
| PRIORITY | 68,7 s | **0,82 s** | 1,12 s |

Ao montar uma tabela FIRST ou PRIORITY, o Praecepta procura regras que nunca disparam: a validação descrita acima. A verificação é exata, e com 1.000 regras aleatórias e sobrepostas ela sai cara: 3,6 s com FIRST, e mais de um minuto com PRIORITY. A avaliação não é afetada, e a tabela real de 119 regras, que é FIRST, é montada em 74 ms. Mesmo assim, um minuto não é aceitável. A [issue #29](https://github.com/brunopacheco1/praecepta/issues/29) acompanha tornar essa verificação opcional, limitada ou mais barata.

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

Isso dá cerca de **100 ns** por decisão na tabela real, numa rodada anterior do mesmo código. Para manter a comparação justa, tudo acima usa entradas `Map` para todos os motores, inclusive o Praecepta.

## Sendo justo: o que os números não dizem

**Onde a comparação favorece o Praecepta, ou tem limites:**
- **O Praecepta faz menos trabalho.** O Camunda e o Drools são interpretadores FEEL de uso geral. Mesmo quando uma célula é um teste simples, eles a avaliam com uma maquinaria capaz de lidar com expressões arbitrárias, e devolvem resultados mais ricos. O Praecepta só suporta o que consegue indexar.
- **Continua sendo um microbenchmark.** Ele roda dentro do processo, numa única máquina. Um serviço real soma HTTP e serialização, que normalmente custam bem mais que 20 µs, então em muitos sistemas o motor nem é o gargalo.
- **Tabelas geradas não são tabelas reais.** As suas regras são aleatórias, então se sobrepõem de jeitos que tabelas reais talvez não se sobreponham. Só uma tabela real foi medida: as tabelas do kit de conformidade do DMN usam todas colunas booleanas, que o Praecepta ainda não suporta.
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
- **Tudo em público:** o código, a CI, o [plano de modernização](https://github.com/brunopacheco1/praecepta/blob/main/docs/MODERNISATION.md) fase a fase, o benchmark com todos os resultados, e as issues. Também o [roadmap](https://github.com/brunopacheco1/praecepta/blob/main/docs/ROADMAP.md), a [arquitetura](https://github.com/brunopacheco1/praecepta/blob/main/docs/ARCHITECTURE.md), o [caso de garantia de segurança](https://github.com/brunopacheco1/praecepta/blob/main/docs/SECURITY-ASSURANCE.md) e a [governança](https://github.com/brunopacheco1/praecepta/blob/main/GOVERNANCE.md), que também diz quem assume se eu parar.
- **Integridade das versões:** cada artefato é assinado com GPG e vem com um SBOM (CycloneDX), proveniência de build (SLSA, verificável com `gh attestation verify`) e avisos de terceiros. Os builds são reproduzíveis.
- **Verificações em cada mudança:** testes em três versões do Java, testes do plugin em três versões do Maven, um estilo de código obrigatório com os avisos do compilador tratados como erros, um quality gate do SonarCloud, CodeQL, verificação de licenças e de vulnerabilidades das dependências, e fuzzing dos dois parsers que leem entradas não confiáveis, em cada pull request que mexe no motor e por mais tempo toda semana.
- **Um `main` protegido:** só commits assinados, e toda mudança chega por um pull request que passa nas verificações e é aprovado por um code owner.
- **Selos de emissores independentes:** o selo **prata** do [OpenSSF Best Practices](https://www.bestpractices.dev/projects/16426) e o nível 1 do OpenSSF Baseline (cada resposta aponta para a sua evidência), a [conformidade REUSE](https://api.reuse.software/info/github.com/brunopacheco1/praecepta) verificada pela FSFE, e o [OpenSSF Scorecard](https://scorecard.dev/viewer/?uri=github.com/brunopacheco1/praecepta), que verifica o repositório de novo toda semana: **8,8 / 10**.

  ![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/brunopacheco1/praecepta/badge)

O item mais baixo do Scorecard é **code review**, e vale explicar. Qualquer pessoa pode fazer um fork do projeto e abrir um pull request. O que essa verificação mede é outra coisa: se cada mudança que chegou ao `main` foi aprovada por alguém além do autor. Hoje isso é zero, porque sou o único mantenedor e ninguém pode aprovar as próprias mudanças. O mesmo motivo impede a nota máxima na proteção de branch: eu faço o merge dos meus próprios pull requests com o bypass de administrador. Cada contribuição que eu revisar aumenta o primeiro número, e um segundo mantenedor completaria os dois. A [governança](https://github.com/brunopacheco1/praecepta/blob/main/GOVERNANCE.md) explica como se tornar um. O que me leva à próxima parte.

## Quer ajudar?

O [site](https://bruno.pacheco.lu/praecepta/) reúne o guia, a documentação da API e como o motor funciona, e o [guia de contribuição](https://github.com/brunopacheco1/praecepta/blob/main/CONTRIBUTING.md) explica como compilar e testar. Boas primeiras issues:

- [intervalos de texto nas células](https://github.com/brunopacheco1/praecepta/issues/13);
- [uma agregação COUNT](https://github.com/brunopacheco1/praecepta/issues/15);
- [rodar o kit de conformidade do DMN](https://github.com/brunopacheco1/praecepta/issues/22).

Revisores e um co-mantenedor também são muito bem-vindos: a [governança](https://github.com/brunopacheco1/praecepta/blob/main/GOVERNANCE.md) descreve os papéis e como as decisões são tomadas.

## O que pode vir a seguir?

Estas são as funcionalidades que acho que seriam úteis; o [roadmap](https://github.com/brunopacheco1/praecepta/blob/main/docs/ROADMAP.md) as coloca em ordem para o próximo ano. **Me diga quais você usaria**: um 👍 na issue já basta, e um comentário se quiser construí-la.

- [Colunas booleanas, de hora e de data-hora](https://github.com/brunopacheco1/praecepta/issues/12)
- [Intervalos de texto nas células](https://github.com/brunopacheco1/praecepta/issues/13)
- [Entradas com vários valores, por exemplo mais de uma nacionalidade](https://github.com/brunopacheco1/praecepta/issues/25)
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

A Trie de 2023 provou a ideia e bateu num limite. Os bitsets a levaram mais longe: nessas tabelas, de 20 a 2.000 vezes mais rápido que motores DMN de uso geral, com 6 a 88 vezes menos memória, e honesto sobre o que não faz, inclusive uma montagem que ainda é lenta para tabelas FIRST e PRIORITY grandes. Experimente, me conte onde ele quebra, e me diga o que você gostaria que ele fizesse a seguir.

**Namárië | See you | Bis geschwënn | Até mais | À bientôt**
