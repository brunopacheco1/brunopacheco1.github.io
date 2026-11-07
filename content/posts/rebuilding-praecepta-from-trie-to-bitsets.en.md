+++
date = 2026-11-07T12:00:00+01:00
title = "Rebuilding My DMN Engine: From a Trie to Bitsets, and a Fair Benchmark Against Camunda and Drools"
description = "Praecepta 0.1.0 is on Maven Central: a DMN decision-table engine rebuilt around bitsets, measured fairly against Camunda 8 and Drools, with its gaps stated plainly. How to use it, how open it is, and what could come next."
categories = ["Coding"]
+++

In January 2023 I wrote about [optimizing decision-making with a trie](/posts/optimizing-decision-making-with-trie-tree/). The idea worked in principle: part of it was implemented, and is still maintained, at [Pictet Tech](https://www.pictet.com/mc/en/about/pictet-tech). The numbers looked great, as you can see in my previous post.

Going through my old repositories for opportunities to test [Claude](https://claude.ai/new), I decided to explore the major problem of that solution: the trie's memory exploded as tables grew; a generated table of 1,000 rules took **over 1 GB**. And during the development process, Claude told me that the first benchmark was not entirely fair, as Praecepta ran a different hit policy from Camunda and Kogito, and two of the services read one input from the wrong field.

So I refactored the engine and the benchmark, and completed the Maven plugin that compiles tables at build time. The result of the refactoring can be found in [Praecepta 0.1.0](https://github.com/brunopacheco1/praecepta) on Maven Central. The principles are still the same: a small, dependency-free Java engine for DMN decision tables. It now also has a [website](https://bruno.pacheco.lu/praecepta/) for the guide and the API documentation.

This post covers what changed, a benchmark against Camunda and Drools that I tried hard to make fair (including where it flatters Praecepta, and where Praecepta is slower), and what those engines do that Praecepta does not. This post also covers how open Praecepta is, and what could come next. That last part is where I would love your opinion.

## From a trie to bitsets

A decision table is a simple list of rules. Each rule tests some input columns (`country is "PT" or "ES"`, `age in [18..65)`, or "anything") and produces outputs. The previous solution, a custom implementation of a [trie](https://www.geeksforgeeks.org/dsa/trie-insert-and-search/), stored every possible path through the columns, and every "anything" cell multiplied the paths. That multiplication was the memory problem!

I tried a couple of things back then, including quantum computing via [qengine](https://github.com/brunopacheco1/qengine/tree/main/src/main/java/com/github/brunopacheco1/qengine): encoding the input as bits and each rule as quantum logic gates, to see whether a quantum circuit could beat the trie. It ran only on a simulator, but the thought stayed with me: rule matching is boolean logic over bits. The bitset index is the classical, practical version of that thought.

In the end, I stopped thinking about it. But now, in 2026, and with Claude's support, the new core uses **bitsets**. Every rule gets one bit. For every column, each distinct value the rules can tell apart gets a mask: the set of rules that accept it.

| Column `country` | rule 1 | rule 2 | rule 3 | rule 4 |
|---|:---:|:---:|:---:|:---:|
| `"PT"` | 1 | 0 | 1 | 1 |
| `"FR"` | 0 | 1 | 1 | 0 |
| any other value | 0 | 0 | 1 | 1 |

Rule 1 tests `"PT"`, rule 2 tests `"FR"`, rule 3 is "anything" (so it is in every mask), and rule 4 is `not("FR")`. Evaluating an input looks up one mask per column and ANDs them together. A 64-bit AND checks 64 rules at once, and the bundled 119-rule table fits in two words. Memory grows with columns × values × rules, never with the product of values, and the 1,000-rule table that needed 1 GB now takes **0.6 MB**, and reading it from DMN and building it takes about a quarter of a second in a fresh JVM.

And the best part: the hit policy falls out of the bits too. Bits are numbered in rule order (or priority order), so FIRST is "lowest set bit", which is a single CPU instruction.

Numbers and dates work the same way: the boundaries the rules mention (`18`, `65`, `date("2025-01-01")`) split each column into intervals, a binary search finds the interval, and each interval has its mask.

## What else changed

The core was not the only change. Praecepta now also has:

- **Strict DMN 1.3 hit policies:** UNIQUE, ANY, PRIORITY, FIRST, OUTPUT ORDER, RULE ORDER, and COLLECT with SUM, MIN and MAX.
- **FEEL cells** on string, number and date columns: lists, `not(...)`, comparisons, intervals and `null`.
- **Validation when the table is built:** overlapping rules, conflicting outputs, rules that can never fire, and *gaps* (inputs no rule covers, with an example).
- **DMN import and export:** Praecepta reads DMN 1.1–1.5 files, including those from Camunda Modeler.
- **A Maven plugin** that validates tables at build time and generates typed input and output records, so your application ships only the engine.
- **Java 21+**, with the engine as a JPMS module. Everything is tested on Java 21, 25 and 27.

## The benchmark

Since the first benchmark was not fair, I rebuilt it from scratch. This time Praecepta runs against Camunda 8 and Drools (the DMN engine inside Kogito), and every engine gets exactly the same work.

### How I measured it

- **Same rules, same file, for every engine.** Each table is written once as a DMN document, and every engine, Praecepta included, reads that document, as an application loading a `.dmn` file would. Before any timing, every engine evaluates the whole workload and its answers are compared with Praecepta built straight from the rules. The run aborts if any engine disagrees.
- **Supported, default setups:** Camunda 8's DMN engine (dmn-scala 1.12.6, the one Zeebe embeds) and Drools 10.2.0's DMN runtime.
- **The same input for all:** a `Map` of values. Praecepta's fastest path (generated classes with enums) is shown separately and labelled as its best case.
- **Many tables:**
  - **Real:** the 119-rule table from 2023 (values blurred, FIRST hit policy).
  - **Generated, `synthetic`:** string columns only.
  - **Generated, `mixed`:** string, number and date columns, with comparisons, intervals and `not(...)`.
  - **Sizes:** the generated tables run from 10 to 10,000 rules and from 5 to 20 columns. The same 1,000 rules are also run under each hit policy, and with inputs that match no rule.
- **More than an average:** latency percentiles, memory allocated per decision, throughput with 1, 4 and 8 threads sharing one engine, the time to build an engine, and the time to the first decision in a fresh JVM.
- **The released engine:** I measured the same engine code that is in the 0.1.0 release.
- **Machine:** Apple M2 Pro (8 performance cores), 16 GB, macOS 26.7.1, OpenJDK 27.0.1. JMH: 5 × 2 s warm-up and 5 × 3 s measurement iterations, 3 forks.
- **Everything is public:** [the results](https://github.com/brunopacheco1/praecepta/tree/main/praecepta-benchmarks/results/2026-11-07-apple-m2-pro-full) keep every log, the raw JMH output and the environment, and one script reruns the whole suite in about 2.5 hours. To check that the picture is not an artefact of my machine, I also ran the short profile on a [GitHub Actions runner](https://github.com/brunopacheco1/praecepta/actions/runs/42391587868) (AMD EPYC, 4 vCPUs, Linux, Java 25). It used one fork, so its numbers are noisy, and its absolute times are 1.5–3× higher, but the ranking holds on every table. On mixed-1000, for example, Praecepta took 1.0 µs, Camunda 519 µs and Drools 421 µs.

### Latency as tables grow

{{< chart >}}
type: 'line',
data: {
  labels: ['10 rules', '100 rules', '1,000 rules', '10,000 rules'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.31, 0.44, 0.64, 1.51], borderColor: '#0e7490', backgroundColor: '#0e7490' },
    { label: 'Camunda 8 (dmn-scala 1.12.6)', data: [7.05, 33.2, 263, 3118], borderColor: '#f97316', backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [7.95, 31.3, 256, 3122], borderColor: '#6366f1', backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Time per evaluation on mixed tables (µs, log scale, lower is better)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'µs' } } }
}
{{< /chart >}}

| µs per evaluation | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| real (119 rules) | **0.19** | 19.3 | 43.5 |
| synthetic-10 | **0.15** | 4.20 | 5.89 |
| synthetic-100 | **0.16** | 19.1 | 14.5 |
| synthetic-1000 | **0.25** | 146 | 126 |
| synthetic-10000 | **0.67** | 1,333 | 1,419 |
| mixed-10 | **0.31** | 7.05 | 7.95 |
| mixed-100 | **0.44** | 33.2 | 31.3 |
| mixed-1000 | **0.64** | 263 | 256 |
| mixed-10000 | **1.51** | 3,118 | 3,122 |

The bigger the table, the bigger the gap:
- **Praecepta:** from 10 to 10,000 mixed rules, its time goes from 0.31 to 1.51 µs, 5× slower for 1,000× more rules.
- **Camunda and Drools:** their time grows roughly with the number of rules, from about 7 µs to about 3,100 µs.
- **The ratio:** Praecepta is 25–40× faster on 10 rules, 70–120× on 100, 400–600× on 1,000 and about 2,000× on 10,000. On the real table it is 104× faster than Camunda and 233× faster than Drools.

More columns cost Praecepta little: with 1,000 rules, going from 6 to 18 mixed columns takes it from 0.62 to 1.12 µs. Number and date columns cost it 2–3× string columns, because a binary search over decimals is slower than a hash lookup; that one has [an issue](https://github.com/brunopacheco1/praecepta/issues/16).

Not every number is equally precise: 51 of the 63 latency measurements have a 99.9% confidence interval within ±10%. The widest is Drools on synthetic-10000, at ±33%. All of them are in the results, if you want to check.

### Hit policies, and inputs that match no rule

| µs per evaluation, 1,000 mixed rules | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| COLLECT | **0.64** | 263 | 256 |
| FIRST | **0.53** | 49.6 | 248 |
| PRIORITY | **0.55** | 287 | 252 |
| RULE ORDER | **0.64** | 285 | 257 |
| UNIQUE (1,000 rules that never overlap) | **0.23** | 161 | 128 |
| No rule matches (COLLECT) | **0.32** | 261 | 197 |

- **Praecepta:** the hit policy barely changes its time, because it is one operation on the bits that matched. An input that matches nothing costs no more than one that matches.
- **Camunda:** it is 5× faster under FIRST, which suggests it stops at the first match.
- **Drools:** it takes about the same time under every policy. When nothing matches, it still pays about three quarters of the time of a match, and Camunda nearly all of it.
- **The real table** has no such inputs: its last rule matches anything.

### Tail latency and memory per decision

| µs per evaluation | p50 | p99 | p99.9 |
|---|---:|---:|---:|
| **Praecepta**, real | **0.17** | **0.29** | **0.54** |
| Camunda 8, real | 17.7 | 32.4 | 41.1 |
| Drools 10.2.0, real | 51.8 | 69.8 | 85.5 |
| **Praecepta**, mixed-1000 | **0.67** | **1.00** | **2.57** |
| Camunda 8, mixed-1000 | 250 | 323 | 444 |
| Drools 10.2.0, mixed-1000 | 260 | 309 | 386 |

- **Tails:** Praecepta's 99.9th percentile on mixed-1000 (2.6 µs) is still about 100× below the others' median.
- **The slowest decisions:** each engine's single slowest decision took up to a few milliseconds, most likely from pauses in the JVM or the operating system, which every engine sees.
- **Allocation:** on tables of up to 1,000 rules, Praecepta allocates 32 to 466 bytes per decision. Camunda allocates 22 KB to 1.5 MB, and Drools 16 KB to 375 KB. Less garbage means less garbage collection under load.

### Throughput with threads

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
  plugins: { title: { display: true, text: 'Decisions per millisecond on the real table, one engine shared by all threads (log scale, higher is better)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'decisions / ms' } } }
}
{{< /chart >}}

Here, all three engines do well: they can be shared between threads, and all of them scale.
- **From 1 to 8 threads:** Praecepta goes from 5,374 to 33,796 decisions per millisecond (6.3×), Camunda from 61 to 451 (7.4×) and Drools from 21 to 146 (7.0×).
- **Locks:** Praecepta's engine is immutable, so threads share it without any.
- **mixed-1000, 8 threads:** 11,721 decisions per millisecond for Praecepta, against 29 for Camunda and 23 for Drools.

### Start-up and building

| ms | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| Read the DMN and build the engine, warm JVM: real | **6.5** | 22.6 | 36.1 |
| Read the DMN and build the engine, warm JVM: mixed-1000 | **45** | 57 | 437 |
| First decision in a fresh JVM: real | **76** | 469 | 379 |
| First decision in a fresh JVM: mixed-1000 | **240** | 774 | 1,005 |

"First decision" starts a new JVM, reads the DMN file, builds the engine and evaluates one input. On the real table, Praecepta answers in 76 ms, 5–6× sooner than the others. That matters for command-line tools, serverless functions and test suites, and much less for long-running services.

### Memory

{{< chart >}}
type: 'bar',
data: {
  labels: ['10 rules', '100 rules', '1,000 rules', '10,000 rules'],
  datasets: [
    { label: 'Praecepta 0.1.0', data: [0.6, 0.6, 1.1, 5.0], backgroundColor: '#0e7490' },
    { label: 'Camunda 8', data: [3.6, 4.8, 16.3, 133], backgroundColor: '#f97316' },
    { label: 'Drools 10.2.0', data: [6.6, 7.4, 15.9, 108], backgroundColor: '#6366f1' }
  ]
},
options: {
  plugins: { title: { display: true, text: 'Retained heap after building, mixed tables (MB, log scale, lower is better)' } },
  scales: { y: { type: 'logarithmic', title: { display: true, text: 'MB' } } }
}
{{< /chart >}}

"Retained heap" is the heap still in use after building the engine in a fresh JVM and running a garbage collection.
- **At 10,000 rules:** Praecepta holds 5 MB, against 133 MB for Camunda and 108 MB for Drools.
- **Across all the generated tables:** it uses 6 to 88 times less.
- **The real table:** Praecepta's footprint there was too small for the measurement to resolve (under 0.5 MB).

### Where Praecepta is slower: building FIRST and PRIORITY tables

| Build time in a fresh JVM, 1,000 mixed rules | Praecepta 0.1.0 | Camunda 8 | Drools 10.2.0 |
|---|---:|---:|---:|
| COLLECT | **0.30 s** | 0.87 s | 1.17 s |
| RULE ORDER | **0.27 s** | 0.83 s | 1.07 s |
| FIRST | 3.6 s | **0.85 s** | 1.15 s |
| PRIORITY | 68.7 s | **0.82 s** | 1.12 s |

When it builds a FIRST or PRIORITY table, Praecepta looks for rules that can never fire: the validation described above. The check is exact, and on 1,000 random, overlapping rules it is expensive: 3.6 s under FIRST, and over a minute under PRIORITY. Evaluation is not affected, and the real 119-rule FIRST table builds in 74 ms. Still, a minute is not acceptable, and fixing it is on my list: [issue #29](https://github.com/brunopacheco1/praecepta/issues/29) tracks making the check optional, bounded or cheaper.

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

That is about **100 ns** per decision on the real table, from an earlier run of the same code. But to keep the comparison fair, everything above uses `Map` inputs for every engine, Praecepta included.

## Being fair: what the numbers do not say

Numbers like these are easy to oversell, so here is what they do not say.

**Where the comparison flatters Praecepta, or is limited:**
- **Praecepta does less work.** Camunda and Drools are general FEEL interpreters. Even when a cell is a simple test, they evaluate it through machinery that can handle arbitrary expressions, and they return richer results. Praecepta only supports what it can index.
- **It is still a microbenchmark.** It runs in-process, on one machine. A real service adds HTTP and serialisation, which usually cost far more than 20 µs, so in many systems the engine is not the bottleneck at all.
- **Generated tables are not real ones.** Their rules are random, so they overlap in ways real tables may not. Only one real table was measured: the DMN conformance kit's tables all use boolean columns, which Praecepta does not support yet.
- **Not every option was measured.** Drools' compiled (executable) model, generated at build time by Kogito, was not measured; it may be faster than the runtime DMN path I used. Drools' experimental "alpha network" option gave wrong answers on these tables in 10.2.0, so the harness refused to time it.

**What Camunda and Drools do that Praecepta does not:**
- **Full FEEL:** functions, arithmetic, contexts, lists, and tests across several columns (`income > 2 * rent`). Praecepta cells are FEEL *simple unary tests* on one column.
- **Decision requirement graphs:** decisions feeding other decisions, business knowledge models and boxed expressions.
- **More types:** boolean, time and date-time columns (planned).
- **Tooling and services:** visual modellers, decision services, process engines, audit and explanation of decisions.
- **Proven conformance:** Drools runs the official DMN conformance kit; Praecepta has not been run against it yet.
- **Maturity:** large communities, years of production use, and commercial support.

So, if your decisions are mostly decision tables with simple cell tests, and latency, memory or start-up matter to you, give Praecepta a try. If you need full DMN, Camunda and Drools are the better choice.

## How to use it

Using it is simple. First, add the engine (it has no dependencies):

```xml
<dependency>
  <groupId>io.github.brunopacheco1</groupId>
  <artifactId>praecepta-engine</artifactId>
  <version>0.1.0</version>
</dependency>
```

Then load a DMN file, bind the inputs, and evaluate:

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
- **Everything in public:** the code, the CI, the [modernisation plan](https://github.com/brunopacheco1/praecepta/blob/main/docs/MODERNISATION.md) phase by phase, the benchmark harness and every result, and the issues. So are the [roadmap](https://github.com/brunopacheco1/praecepta/blob/main/docs/ROADMAP.md), the [architecture](https://github.com/brunopacheco1/praecepta/blob/main/docs/ARCHITECTURE.md), the [security assurance case](https://github.com/brunopacheco1/praecepta/blob/main/docs/SECURITY-ASSURANCE.md) and the [governance](https://github.com/brunopacheco1/praecepta/blob/main/GOVERNANCE.md), which also says who takes over if I stop.
- **Release integrity:** every artifact is GPG-signed and ships with an SBOM (CycloneDX), build provenance (SLSA, verifiable with `gh attestation verify`) and third-party notices. The builds are reproducible.
- **Checks on every change:** tests on three Java versions, plugin tests on three Maven versions, an enforced code style with compiler warnings as errors, a SonarCloud quality gate, CodeQL, dependency licence and vulnerability checks, and fuzzing of the two parsers that read untrusted input, on every pull request that touches the engine and for longer every week.
- **A protected `main`:** signed commits only, and every change arrives through a pull request that passes the checks and is approved by a code owner.
- **Badges from independent issuers:** the [OpenSSF Best Practices](https://www.bestpractices.dev/projects/16426) **silver** badge and OpenSSF Baseline level 1 (every answer links to its evidence), [REUSE compliance](https://api.reuse.software/info/github.com/brunopacheco1/praecepta) checked by the FSFE, and the [OpenSSF Scorecard](https://scorecard.dev/viewer/?uri=github.com/brunopacheco1/praecepta), which rechecks the repository every week: **8.8 / 10**.

  ![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/brunopacheco1/praecepta/badge)

The lowest Scorecard item is **code review**, and it is worth explaining. Anyone can fork the project and open a pull request. The check measures something else: whether each change that reached `main` was approved by someone other than its author. Today that is zero, because I am the only maintainer, and nobody can approve their own changes. The same reason keeps branch protection from a full score: I merge my own pull requests with an administrator's bypass. Every contribution I review raises the first, and a second maintainer would close both. The [governance](https://github.com/brunopacheco1/praecepta/blob/main/GOVERNANCE.md) explains how to become one. Which brings me to the next part.

## Want to help?

If you would like to contribute, the [website](https://bruno.pacheco.lu/praecepta/) gathers the guide, the API documentation and how the engine works, and the [contributing guide](https://github.com/brunopacheco1/praecepta/blob/main/CONTRIBUTING.md) explains how to build and test it. Here are some good first issues:

- [string ranges in cells](https://github.com/brunopacheco1/praecepta/issues/13);
- [a COUNT aggregation](https://github.com/brunopacheco1/praecepta/issues/15);
- [running the DMN conformance kit](https://github.com/brunopacheco1/praecepta/issues/22).

I would also welcome reviewers and a co-maintainer: the [governance](https://github.com/brunopacheco1/praecepta/blob/main/GOVERNANCE.md) describes the roles and how decisions are made.

## What could come next?

These are the features I think would be useful; the [roadmap](https://github.com/brunopacheco1/praecepta/blob/main/docs/ROADMAP.md) puts them in order for the next year. **Tell me which ones you would use**: a 👍 on the issue is enough, and a comment if you would like to build it.

- [Boolean, time and date-time columns](https://github.com/brunopacheco1/praecepta/issues/12)
- [String ranges in cells](https://github.com/brunopacheco1/praecepta/issues/13)
- [Inputs with several values, e.g. more than one nationality](https://github.com/brunopacheco1/praecepta/issues/25)
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

The 2023 trie proved the idea and hit a wall. Bitsets took it further: on these tables, 20 to 2,000 times faster than general-purpose DMN engines, with 6 to 88 times less memory, and honest about what it does not do, including a build that is still slow for large FIRST and PRIORITY tables. Give it a try, tell me where it breaks, and tell me what you would like it to do next.

**Namárië | See you | Bis geschwënn | Até mais | À bientôt**
