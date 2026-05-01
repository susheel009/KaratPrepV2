# Java Fundamentals — Complete Karat Interview Syllabus

> **Purpose:** 100% coverage of Java fundamentals, Spring basics, and Kafka basics for Karat screening.
> Each topic file contains: rapid-fire Q&A, deeper explanations, code snippets, and the "Definition → Mechanism → Implication" talk track format.

---

## How to use

1. **Rapid-fire drill:** Read each Q, cover the A, answer aloud in ≤30 seconds.
2. **Deep-dive:** If you stumble, read the explanation below each Q&A.
3. **Self-check:** Every file ends with a "can you answer these cold?" checklist.

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
