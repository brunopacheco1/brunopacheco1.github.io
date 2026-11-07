+++
date = 2026-11-07T12:00:00+01:00
title = "Rebuilding Praecepta: From a Trie to Bitsets, and a Fair Benchmark Against Camunda and Drools"
description = "Praecepta 0.1.0 is on Maven Central: a DMN decision-table engine rebuilt around bitsets, measured fairly against Camunda 8 and Drools, with its gaps stated plainly. How to use it, how open it is, and what could come next."
categories = ["Coding"]
+++

In January 2023 I wrote about [replacing DMN engines with a trie](/posts/optimizing-decision-making-with-trie-tree/). The idea worked, and the numbers looked great. Looking back at it this year, I found two problems. The trie's memory exploded as tables grew: a generated table of 1,000 rules took **over 1 GB**. And the benchmark was not fair: Praecepta ran a different hit policy from Camunda and Kogito, and two of the services read one input from the wrong field.

So I rebuilt it. [Praecepta 0.1.0](https://github.com/brunopacheco1/praecepta) is now on Maven Central: a small, dependency-free Java engine for DMN decision tables. This post covers what changed, a benchmark against Camunda and Drools that I tried hard to make fair (including where it flatters Praecepta), what those engines do that Praecepta does not, how open the project is, and what could come next. That last part is where I would love your opinion.

## From a trie to bitsets

A decision table is a list of rules. Each rule tests some input columns (`country is "PT" or "ES"`, `age in [18..65)`, or "anything") and produces outputs. The trie stored every possible path through the columns, and every "anything" cell multiplied the paths. That multiplication was the memory problem, and no amount of tuning fixes it.

The new core uses **bitsets**. Every rule gets one bit. For every column, each distinct value the rules can tell apart gets a mask: the set of rules that accept it.

| Column `country` | rule 1 | rule 2 | rule 3 | rule 4 |
|---|:---:|:---:|:---:|:---:|
| `"PT"` | 1 | 0 | 1 | 1 |
| `"FR"` | 0 | 1 | 1 | 0 |
| any other value | 0 | 0 | 1 | 1 |

Rule 1 tests `"PT"`, rule 2 tests `"FR"`, rule 3 is "anything" (so it is in every mask), and rule 4 is `not("FR")`. Evaluating an input looks up one mask per column and ANDs them together. A 64-bit AND checks 64 rules at once, and the bundled 119-rule table fits in two words. Memory grows with columns × values × rules, never with the product of values, and the 1,000-rule table that needed 1 GB now takes **0.4 MB** and builds in about 80 ms.

The hit policy falls out of the bits too. Bits are numbered in rule order (or priority order), so FIRST is "lowest set bit", which is a single CPU instruction.

Numbers and dates work the same way: the boundaries the rules mention (`18`, `65`, `date("2025-01-01")`) split each column into intervals, a binary search finds the interval, and each interval has its mask.

## What else changed

- **Strict DMN 1.3 hit policies:** UNIQUE, ANY, PRIORITY, FIRST, OUTPUT ORDER, RULE ORDER, and COLLECT with SUM, MIN and MAX.
- **FEEL cells** on string, number and date columns: lists, `not(...)`, comparisons, intervals and `null`.
- **Validation when the table is built:** overlapping rules, conflicting outputs, rules that can never fire, and *gaps* (inputs no rule covers, with an example).
- **DMN import and export:** Praecepta reads DMN 1.1–1.5 files, including those from Camunda Modeler.
- **A Maven plugin** that validates tables at build time and generates typed input and output records, so your application ships only the engine.
- **Java 21+**, with the engine as a JPMS module. Everything is tested on Java 21, 25 and 27.

## The benchmark

### How it was measured
- **Same rules for every engine.** One rule list is written as DMN for Camunda and Drools. Before any timing, every engine evaluates the whole workload and the results are compared; the benchmark aborts if any engine disagrees.
- **Supported, default setups:** Camunda 8's DMN engine (dmn-scala 1.12.6, the one Zeebe embeds) and Drools 10.2.0's DMN runtime. Each engine parses its model once; only evaluation is timed.
- **The same input for all:** a `Map` of values. Praecepta's fastest path (generated classes with enums) is shown separately and labelled as its best case.
- **Five tables:** the real 119-rule table from 2023 (values blurred), generated string tables of 100 and 1,000 rules, and generated tables of 100 and 1,000 rules that mix string, number and date columns with comparisons, intervals and `not(...)`.
- **Released code:** Praecepta 0.1.0 straight from Maven Central.
- **Machine:** Apple M2 Pro, 16 GB, macOS 26.7, OpenJDK 27. JMH: 5 × 2 s warm-up and 5 × 3 s measurement iterations, 2 forks, one thread.

### Latency

{{< chart >}}
type: 'bar',
data: {
  labels: ['real (119 rules)', 'synthetic-100', 'synthetic-1000', 'mixed-100', 'mixed-1000'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.188, 0.150, 0.226, 0.485, 0.684], backgroundColor: '#0e7490' },
    { label: 'Camunda 8 (dmn-scala 1.12.6)', data: [16.70, 19.47, 127.0, 37.21, 282.8], backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [59.79, 16.38, 143.9, 32.38, 276.1], backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Time per evaluation (µs, log scale, lower is better)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'µs' } } }
}
{{< /chart >}}

| µs per evaluation (± 99.9% error) | real | synthetic-100 | synthetic-1000 | mixed-100 | mixed-1000 |
|---|---:|---:|---:|---:|---:|
| **Praecepta 0.1.0** | **0.19** ± 0.01 | **0.15** ± 0.01 | **0.23** ± 0.01 | **0.49** ± 0.02 | **0.68** ± 0.04 |
| Camunda 8 | 16.7 ± 0.1 | 19.5 ± 0.2 | 127 ± 3 | 37.2 ± 2.2 | 283 ± 25 |
| Drools 10.2.0 | 59.8 ± 6.1 | 16.4 ± 2.7 | 144 ± 42 | 32.4 ± 1.2 | 276 ± 17 |

Praecepta is between **70×** (mixed-100 against Drools) and **640×** (synthetic-1000 against Drools) faster. Notice two things. First, Praecepta's time barely grows from 100 to 1,000 rules, while the others grow roughly linearly. Second, number and date columns cost Praecepta 2–3× string columns, because a binary search over decimals is slower than a hash lookup; that one has [an issue](https://github.com/brunopacheco1/praecepta/issues/16).

Drools was the noisiest engine (±42 µs on synthetic-1000). To check that the picture is not an artefact of my machine, I ran the same suite on a GitHub Actions runner (x86, Java 25). The absolute numbers differ, but the ranking holds: Praecepta 0.15–0.65 µs; Camunda 13.7–274 µs; Drools 12.4–218 µs.

### Memory and build time

{{< chart >}}
type: 'bar',
data: {
  labels: ['real (119 rules)', 'synthetic-100', 'synthetic-1000', 'mixed-100', 'mixed-1000'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.2, 0.3, 0.4, 0.4, 0.9], backgroundColor: '#0e7490' },
    { label: 'Camunda 8', data: [4.6, 4.9, 16.1, 5.0, 16.5], backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [6.6, 7.0, 13.8, 7.5, 15.8], backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Retained heap after building (MB, lower is better)' } },
  scales: { y: { title: { display: true, text: 'MB' } } }
}
{{< /chart >}}

| Build time | real | synthetic-100 | synthetic-1000 | mixed-100 | mixed-1000 |
|---|---:|---:|---:|---:|---:|
| **Praecepta 0.1.0** | **39 ms** | **60 ms** | **82 ms** | **68 ms** | **167 ms** |
| Camunda 8 | 437 ms | 488 ms | 700 ms | 534 ms | 808 ms |
| Drools 10.2.0 | 336 ms | 401 ms | 628 ms | 491 ms | 1,060 ms |

Each engine is built in a fresh JVM, and "retained heap" is the heap still in use after building it and running a garbage collection.

### Praecepta's best case

Feeding a `Map` costs two hash lookups per column: one to read the value, one to find its mask. With the Maven plugin, inputs are generated records, and string columns with a value list become enums, whose masks are found by position instead of by hashing:

{{< chart >}}
type: 'bar',
data: {
  labels: ['Map of strings', 'generated record (strings)', 'generated record (enums)'],
  datasets: [{ label: 'ns per evaluation, real table', data: [231, 142, 101], backgroundColor: ['#94a3b8', '#38bdf8', '#0e7490'] }]
},
options: {
  plugins: { title: { display: true, text: 'Praecepta input paths on the real table (ns, lower is better)' }, legend: { display: false } }
}
{{< /chart >}}

That is about **100 ns** per decision on the real table. To keep the comparison fair, the engine comparison above uses `Map` inputs for every engine, Praecepta included.

## Being fair: what the numbers do not say

**Where the comparison flatters Praecepta:**
- **Praecepta does less work.** Camunda and Drools are general FEEL interpreters. Even when a cell is a simple test, they evaluate it through machinery that can handle arbitrary expressions, and they return richer results. Praecepta only supports what it can index.
- **The test is narrow.** It runs in-process, single-threaded, on a warm JVM. A real service adds HTTP and serialisation, which usually cost far more than 20 µs, so in many systems the engine is not the bottleneck at all.
- **Not every option was measured.** Drools' compiled (executable) model, generated at build time by Kogito, was not measured; it may be faster than the runtime DMN path I used. Drools' experimental "alpha network" option gave wrong answers on these tables in 10.2.0, so the harness refused to time it.

**What Camunda and Drools do that Praecepta does not:**
- **Full FEEL:** functions, arithmetic, contexts, lists, and tests across several columns (`income > 2 * rent`). Praecepta cells are FEEL *simple unary tests* on one column.
- **Decision requirement graphs:** decisions feeding other decisions, business knowledge models and boxed expressions.
- **More types:** boolean, time and date-time columns (planned).
- **Tooling and services:** visual modellers, decision services, process engines, audit and explanation of decisions.
- **Proven conformance:** Drools runs the official DMN conformance kit; Praecepta has not been run against it yet.
- **Maturity:** large communities, years of production use, and commercial support.

If your decisions are mostly decision tables with simple cell tests, and latency, memory or start-up matter, Praecepta is worth a look. If you need full DMN, use Camunda or Drools.

## How to use it

Add the engine (it has no dependencies):

```xml
<dependency>
  <groupId>io.github.brunopacheco1</groupId>
  <artifactId>praecepta-engine</artifactId>
  <version>0.1.0</version>
</dependency>
```

Load a DMN file, bind the inputs, evaluate:

```java
record Customer(String country, Integer age) {}

DecisionTable table;
try (var dmn = Files.newInputStream(Path.of("discount.dmn"))) {
    table = DmnReader.read(dmn);
}
var engine = PraeceptaEngine.<Customer, String>builder()
        .table(table)                              // hit policy, column types, value lists, rules
        .input("country", Customer::country)
        .numberInput("age", Customer::age)
        .output(output -> output.values().get("tier"))
        .build();

engine.evaluate(new Customer("PT", 30));          // e.g. [gold]
engine.validationReport();                         // overlaps, unreachable rules, gaps
```

Or let the Maven plugin compile your tables (Excel, CSV or DMN) at build time:

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

The build fails if the table has errors (for example, two UNIQUE rules that overlap), and only the engine is needed at runtime. The [README](https://github.com/brunopacheco1/praecepta#readme) has the details.

## How open is it?

I wanted a small project that is easy to trust. Here is what that means in practice:

- **Licence:** Apache-2.0. The repository is [REUSE](https://reuse.software)-compliant: every file says who wrote it and under which licence.
- **Everything in public:** the code, the CI, the [modernisation plan](https://github.com/brunopacheco1/praecepta/blob/main/docs/MODERNISATION.md) phase by phase, the benchmark harness, and the issues.
- **Release integrity:** every artifact is GPG-signed and ships with an SBOM (CycloneDX), build provenance (SLSA, verifiable with `gh attestation verify`) and third-party notices. The builds are reproducible.
- **Checks on every change:** tests on three Java versions, plugin tests on three Maven versions, a SonarCloud quality gate, CodeQL, dependency licence and vulnerability checks, and fuzzing of the two parsers that read untrusted input.
- **[OpenSSF Scorecard](https://scorecard.dev/viewer/?uri=github.com/brunopacheco1/praecepta): 7.8 / 10.**

  ![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/brunopacheco1/praecepta/badge)

The lowest Scorecard item is **code review**, and it is worth explaining. Anyone can fork the project and open a pull request. The check measures something else: whether each change that reached `main` was approved by someone other than its author. Today that is zero, because I am the only maintainer, and nobody can approve their own changes. Every contribution I review raises it, and a second maintainer would close it. Which brings me to the next part.

## Want to help?

The [contributing guide](https://github.com/brunopacheco1/praecepta/blob/main/CONTRIBUTING.md) explains how to build and test it. Good first issues:

- [string ranges in cells](https://github.com/brunopacheco1/praecepta/issues/13);
- [a COUNT aggregation](https://github.com/brunopacheco1/praecepta/issues/15);
- [running the DMN conformance kit](https://github.com/brunopacheco1/praecepta/issues/22).

I would also welcome reviewers and a co-maintainer.

## What could come next?

These are the features I think would be useful. **Tell me which ones you would use**: a 👍 on the issue is enough, and a comment if you would like to build it.

- [Boolean, time and date-time columns](https://github.com/brunopacheco1/praecepta/issues/12)
- [String ranges in cells](https://github.com/brunopacheco1/praecepta/issues/13)
- [Decision requirement graphs (chained decisions)](https://github.com/brunopacheco1/praecepta/issues/14)
- [A COUNT aggregation](https://github.com/brunopacheco1/praecepta/issues/15)
- [Faster number and date columns](https://github.com/brunopacheco1/praecepta/issues/16)
- [Compressed bitmaps for very large tables](https://github.com/brunopacheco1/praecepta/issues/17)
- [A Gradle plugin](https://github.com/brunopacheco1/praecepta/issues/18)
- [GraalVM native-image support](https://github.com/brunopacheco1/praecepta/issues/19)
- [A Spring Boot starter](https://github.com/brunopacheco1/praecepta/issues/20)
- [An online playground](https://github.com/brunopacheco1/praecepta/issues/21)
- [Running the DMN conformance kit](https://github.com/brunopacheco1/praecepta/issues/22)

## Wrapping up

The 2023 trie proved the idea and hit a wall. Bitsets took it further: on these tables, 70 to 640 times faster than general-purpose DMN engines, with 18 to 40 times less memory, and honest about what it does not do. Give it a try, tell me where it breaks, and tell me what you would like it to do next.

**See you | Bis geschwënn | Até mais | À bientôt**
