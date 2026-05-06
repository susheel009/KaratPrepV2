# Java Fundamentals — Complete Karat Interview Syllabus

> **Purpose:** 100% coverage of Java fundamentals, Spring basics, and Kafka basics for Karat screening.

## File structure (rewritten files)

Each rewritten topic file follows this incremental order so you build intuition before code:

1. **The Problem** — story-style, no code
2. **Walkthrough** — concept + tiny example, pseudo-code only
3. **First Code** — minimal working example, every non-obvious line commented
4. **Build Up** — practical patterns, one at a time (problem → fix → why)
5. **Going Deep** — interview-level material (memory model, JIT, edge cases)
6. **Memory Aids** — mnemonics, decision trees, "if asked X, think Y"
7. **Cheat Sheet** — rapid-fire Q&A for pre-interview review
8. **Self-Test** — easy → medium → hard
9. **Glossary** — plain-English definitions

## How to use

- **First pass (study):** read sections 1–4 in order, do the §8 self-test.
- **Drill (recall):** cover the answers in §7, recite aloud in ≤30 seconds.
- **15 min before interview:** §7 cheat sheet + §6 memory aids only.

## Doubts workflow

- Capture per-topic questions in `v2/doubts/<NN>_<topic>_doubts.md` (or run `/add-doubt <topic> <question>`).
- Run `/update-files [topic]` to have study files rewritten using your doubts as the calibration signal — addressed doubts get marked `✅ addressed in study file (date)`.
- Run `/push-doubts` to commit `v2/doubts/` and push to `origin/develop`.

## Rewrite status

All 17 files rewritten in the new template.

| # | File | Status | Lines |
|:-:|------|:--:|--:|
| 01 | [01_concurrency.md](./01_concurrency.md) | ✅ | 1018 |
| 02 | [02_collections.md](./02_collections.md) | ✅ | 659 |
| 03 | [03_oop_language.md](./03_oop_language.md) | ✅ | 700 |
| 04 | [04_exceptions.md](./04_exceptions.md) | ✅ | 586 |
| 05 | [05_strings.md](./05_strings.md) | ✅ | 522 |
| 06 | [06_java8_plus.md](./06_java8_plus.md) | ✅ | 593 |
| 07 | [07_jvm.md](./07_jvm.md) | ✅ | 519 |
| 08 | [08_design_patterns.md](./08_design_patterns.md) | ✅ | 618 |
| 09 | [09_solid.md](./09_solid.md) | ✅ | 519 |
| 10 | [10_testing.md](./10_testing.md) | ✅ | 660 |
| 11 | [11_spring_core.md](./11_spring_core.md) | ✅ | 609 |
| 12 | [12_spring_boot.md](./12_spring_boot.md) | ✅ | 690 |
| 13 | [13_spring_rest.md](./13_spring_rest.md) | ✅ | 666 |
| 14 | [14_spring_aop_data.md](./14_spring_aop_data.md) | ✅ | 693 |
| 15 | [15_kafka.md](./15_kafka.md) | ✅ | 661 |
| 16 | [16_debugging.md](./16_debugging.md) | ✅ | 632 |
| 17 | [17_debugging_problems.md](./17_debugging_problems.md) | ✅ (practice set; nav header only) | 675 |

> Note on 17: it's a 20-problem practice set, not a study file — it doesn't fit the 9-section template, so only the comprehensive nav header was added; the existing problems and scoring guide are preserved.

---

## Table of Contents

### Part 1 — Core Java (Sections 01–10)

| # | File | Topic | Priority |
|:-:|------|-------|:--------:|
| 01 | [Concurrency & Threading](./01_concurrency.md) | synchronized, volatile, Atomic*, JMM, happens-before, locks, thread pools, CompletableFuture | 🔴 Critical |
| 02 | [Collections Internals](./02_collections.md) | HashMap internals, ConcurrentHashMap, ArrayList vs LinkedList, TreeMap, equals/hashCode | 🔴 Critical |
| 03 | [OOP & Language](./03_oop_language.md) | Interface vs abstract, overloading vs overriding, final, static, generics, type erasure, PECS | 🔴 Critical |
| 04 | [Exception Handling](./04_exceptions.md) | Checked vs unchecked, try-with-resources, custom exceptions, best practices | 🟡 High |
| 05 | [Strings & Immutability](./05_strings.md) | String pool, StringBuilder, immutability, equals vs ==, intern() | 🟡 High |
| 06 | [Java 8+ Features](./06_java8_plus.md) | Streams, lambdas, Optional, functional interfaces, method references, records, sealed classes | 🟡 High |
| 07 | [JVM Internals](./07_jvm.md) | Heap/stack, GC algorithms, class loading, memory model, JIT | 🟢 Medium |
| 08 | [Design Patterns](./08_design_patterns.md) | Singleton, Factory, Builder, Strategy, Observer, Proxy, Template Method | 🟢 Medium |
| 09 | [SOLID & Design Principles](./09_solid.md) | SRP, OCP, LSP, ISP, DIP, composition vs inheritance, DRY, KISS | 🟢 Medium |
| 10 | [Testing](./10_testing.md) | JUnit 5, Mockito, TDD basics, integration tests, test doubles | 🟢 Medium |

### Part 2 — Spring Ecosystem (Sections 11–14)

| # | File | Topic | Priority |
|:-:|------|-------|:--------:|
| 11 | [Spring Core & DI](./11_spring_core.md) | IoC, DI types, bean lifecycle, scopes, @Autowired, @Qualifier, profiles, properties | 🟡 High |
| 12 | [Spring Boot](./12_spring_boot.md) | Auto-config, starters, Actuator, @Transactional, @Async, scheduling, error handling | 🟡 High |
| 13 | [Spring REST & Web](./13_spring_rest.md) | @RestController, request mapping, validation, exception handling, security basics | 🟡 High |
| 14 | [Spring AOP & Data](./14_spring_aop_data.md) | AOP concepts, JPA/Hibernate, transactions, N+1, caching | 🟢 Medium |

### Part 3 — Kafka (Section 15)

| # | File | Topic | Priority |
|:-:|------|-------|:--------:|
| 15 | [Kafka Fundamentals](./15_kafka.md) | Architecture, partitions, consumer groups, exactly-once, Spring Kafka | 🟡 High |

### Part 4 — Debugging & Code Reading (Sections 16–17)

| # | File | Topic | Priority |
|:-:|------|-------|:--------:|
| 16 | [Bug Taxonomy & Debugging](./16_debugging.md) | Off-by-one, null handling, race conditions, mutation during iteration, dry-run discipline | 🔴 Critical |
| 17 | [Debugging Problems (20)](./17_debugging_problems.md) | 20 code samples with bugs — attempt, then evaluate | 🔴 Practice |

---

## Priority Legend

| Icon | Meaning | Karat frequency |
|:----:|---------|:---------------:|
| 🔴 | Must know cold — will be asked | ~60% of questions |
| 🟡 | Should know well — likely to come up | ~30% of questions |
| 🟢 | Good to know — occasional or follow-up | ~10% of questions |

---

## Quick Links

- [← Back to v2 Prep Plan](../v2%20Prep%20Plan.md)
- [Level 1 Topics](../level-1/)
- [Level 2 Topics](../level-2/)
- [Level 3 Topics](../level-3/)
