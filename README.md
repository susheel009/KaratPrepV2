# KaratPrepV2 — Complete Java Interview Preparation

A comprehensive, structured guide for mastering Java fundamentals, Spring ecosystem, and Kafka basics for **Citi/Karat interviews** and **senior-level Java engineering**.

## 📌 Overview

**KaratPrepV2** is a complete interview preparation curriculum covering:
- ✅ **17 core Java topics** (01–10: Core Java | 11–14: Spring | 15: Kafka | 16–17: Debugging)
- ✅ **100+ Q&A pairs** with mechanism explanations and code examples
- ✅ **4 companion practice sets** (debugging problems, ladder problems, solutions)
- ✅ **Data structures deep-dive** (arrays, lists, maps, queues, trees, graphs, heaps)
- ✅ **Prioritised learning** (🔴 Critical ~60% of questions | 🟡 High ~30% | 🟢 Medium ~10%)

**Target audience:** Candidates preparing for Citi tech screens, Karat interviews, or anyone aspiring to senior/staff-level Java roles.

---

## 🗂️ Project Structure

```
KaratPrepV2/
├── README.md                              ← You are here
├── v2 Prep Plan.md                        ← Study roadmap & philosophy
│
└── v2/                                    ← Main curriculum
    ├── 00_INDEX.md                        ← Master index & syllabus
    │
    ├── Part 1 — Core Java (Critical Path)
    │   ├── 01_concurrency.md              🔴 Threads, locks, CompletableFuture, JMM
    │   ├── 02_collections.md              🔴 HashMap, ConcurrentHashMap, ArrayList, design
    │   ├── 03_oop_language.md             🔴 Interfaces, inheritance, records, sealed classes
    │   ├── 04_exceptions.md               🟡 Try-with-resources, custom exceptions, best practices
    │   ├── 05_strings.md                  🟡 Immutability, String pool, StringBuilder
    │   ├── 06_java8_plus.md               🟡 Streams, lambdas, Optional, records, switch expressions
    │   ├── 07_jvm.md                      🟢 Heap/stack, GC, JIT, class loading
    │   ├── 08_design_patterns.md          🟢 Singleton, Factory, Builder, Proxy, Observer
    │   ├── 09_solid.md                    🟢 SRP, OCP, LSP, ISP, DIP, DRY, KISS, YAGNI
    │   └── 10_testing.md                  🟢 JUnit 5, Mockito, TDD, test doubles
    │
    ├── Part 2 — Spring Ecosystem
    │   ├── 11_spring_core.md              🟡 IoC, DI types, bean lifecycle, scopes, profiles
    │   ├── 12_spring_boot.md              🟡 Auto-config, starters, @Transactional, @Async
    │   ├── 13_spring_rest.md              🟡 @RestController, validation, error handling, security
    │   └── 14_spring_aop_data.md          🟢 AOP, JPA/Hibernate, transactions, N+1, caching
    │
    ├── Part 3 — Kafka
    │   └── 15_kafka.md                    🟡 Architecture, partitions, consumer groups, exactly-once
    │
    ├── Part 4 — Debugging & Code Reading
    │   ├── 16_debugging.md                🔴 Bug taxonomy, dry-run discipline, concurrency traps
    │   └── 17_debugging_problems.md       🔴 20 predict-the-output & find-the-bug problems
    │
    ├── DataStructures/
    │   └── DataStructures.md              📊 Deep-dive: arrays, lists, maps, queues, trees, graphs
    │
    └── Practice/
        ├── karat-practice-INDEX.md        📋 Master index of practice problems
        ├── karat-debugging-problems.md    🐛 20 debugging problems (DBG-01 through DBG-23)
        ├── karat-ladder-problems.md       🪜 10 OOP debug-and-extend exercises (LADDER-01–10)
        └── karat-solutions.md             ✅ Full solutions for all practice problems
```

---

## 🎯 Quick Start (30 minutes)

**Do this first:**

1. **Read the syllabus**: [`v2/00_INDEX.md`](v2/00_INDEX.md) — 2 min read
2. **Understand the structure**: This README (you're reading it!)
3. **Pick your starting point** below ↓

---

## 📚 How to Use This Guide

### Learning Paths

#### Path 1: I have 2 weeks (Intensive)
- **Week 1 Day 1–2**: Core Java [01–03](#core-java---critical-path)
- **Week 1 Day 3**: Collections [02](#02--collections-internals) deep-dive
- **Week 1 Day 4**: Concurrency [01](#01--concurrency--threading) drills
- **Week 1 Day 5**: Spring [11–13](#spring-ecosystem)
- **Week 2 Day 1**: Kafka [15](#15--kafka-fundamentals-high)
- **Week 2 Day 2**: Debugging [16](#16--bug-taxonomy--debugging-critical) + practice
- **Week 2 Day 3–4**: Timed ladder problems [Practice](#practice--companion-drill-sets)

#### Path 2: I have 4 weeks (Balanced)
- **Weeks 1–2**: Core Java topics [01–10](#core-java---critical-path) (one per day)
- **Week 3**: Spring [11–14](#spring-ecosystem) + Kafka [15](#15--kafka-fundamentals-high)
- **Week 4**: Debugging [16–17](#debugging--code-reading) + all practice problems under timed conditions

#### Path 3: I want to focus only on Critical items (1 week)
- **Day 1**: [Concurrency (01)](#01--concurrency--threading)
- **Day 2**: [Collections (02)](#02--collections-internals)
- **Day 3**: [OOP Language (03)](#03--oop--language-fundamentals) + [Strings (05)](#05--strings--immutability)
- **Day 4**: [Debugging (16–17)](#debugging--code-reading)
- **Days 5–7**: Practice problems, re-read and drill

#### Path 4: Interview is in 3 days (Emergency Mode)
1. Drill critical Q&A from [01](#01--concurrency--threading), [02](#02--collections-internals), [03](#03--oop--language-fundamentals), [16](#16--bug-taxonomy--debugging-critical) aloud (2 hours)
2. Dry-run the first 5 debugging problems at speed (30 min)
3. Run one full ladder problem under 35-min timer (35 min)
4. Sleep. Repeat the morning of the interview.

---

## 🔴 Core Java — Critical Path

These topics appear in **~60% of Karat questions**. Master these before others.

### 01 — Concurrency & Threading
**Priority: 🔴 Critical**

**Topics covered:**
- Thread lifecycle (`NEW → RUNNABLE → BLOCKED/WAITING → TERMINATED`)
- `synchronized`: monitor locks, reentrancy, happens-before
- `volatile`: visibility guarantees, atomicity limitations
- Java Memory Model (JMM) & happens-before rules
- `AtomicInteger`, `AtomicLong`, `LongAdder` (CAS-based lock-free)
- `ReentrantLock`, `ReadWriteLock`, `StampedLock`
- `ExecutorService` & thread pool sizing
- `CompletableFuture` async composition
- Synchronisers: `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`
- `ThreadLocal` & cleanup patterns
- `ForkJoinPool` for recursive divide-and-conquer
- Deadlock: four conditions, prevention strategies
- Virtual threads (Java 21+)

**Key Q&A:**
- Explain `synchronized` — what it locks, why it's reentrant
- `volatile` guarantees vs limitations — give the `x++` example
- CAS — what it is, where Java uses it (AtomicInteger)
- Four deadlock conditions and how to break each one
- `Future` vs `CompletableFuture` — write a chain from memory
- Thread pool sizing: CPU-bound = `cores+1`, I/O-bound = `cores × (1 + wait/compute)`

**Code patterns:**
- Double-checked locking singleton with `volatile`
- CompletableFuture chaining: `supplyAsync → thenApply → thenCompose → exceptionally`
- Producer-consumer with `BlockingQueue` & poison pill

**File:** [`v2/01_concurrency.md`](v2/01_concurrency.md)

---

### 02 — Collections Internals
**Priority: 🔴 Critical**

**Topics covered:**
- Collection hierarchy (List, Set, Queue, Map)
- `HashMap` internals: hashing, bucket collision, red-black trees, rehashing
- `equals`/`hashCode` contract & mutable keys
- `ConcurrentHashMap`: Java 7 segments → Java 8 per-bucket locking
- Atomic compound operations: `computeIfAbsent`, `compute`, `merge`
- `ArrayList` vs `LinkedList`: cache locality, when ArrayList almost always wins
- `TreeMap` (red-black tree): O(log n) operations, range queries
- `LinkedHashMap`: insertion order & LRU cache
- `EnumSet` & `EnumMap`: bitfield-backed efficiency
- Immutable collections (Java 9+): `List.of`, `Set.of`, `Map.of`
- Fail-fast (`ArrayList`, `HashMap`) vs fail-safe (`CopyOnWriteArrayList`, `ConcurrentHashMap`)

**Key Q&A:**
- Walk through `HashMap.put()` — hashing, bucket index, collision, treeification
- Four-way comparison: HashMap/Hashtable/synchronizedMap/ConcurrentHashMap
- Why `ConcurrentHashMap` is faster than `synchronizedMap`
- `ArrayList` vs `LinkedList` — cache locality argument
- What changed in `ConcurrentHashMap` between Java 7 and 8
- `equals`/`hashCode` contract — what breaks if violated

**Code patterns:**
- ConcurrentHashMap atomic operation: `map.computeIfAbsent(key, k -> compute(k))`
- LRU cache with `LinkedHashMap` in 5 lines
- Immutable defensive copy: `List<String> copy = List.copyOf(mutableList)`

**File:** [`v2/02_collections.md`](v2/02_collections.md)

---

### 03 — OOP & Language Fundamentals
**Priority: 🔴 Critical**

**Topics covered:**
- Four pillars of OOP: encapsulation, abstraction, inheritance, polymorphism
- Interface vs abstract class (Java 8+ default methods)
- Method overloading (compile-time dispatch) vs overriding (runtime dispatch)
- Generics & type erasure: what's lost at runtime, what can't you do
- PECS (Producer Extends, Consumer Super)
- `final`: variable (immutability), method (no override), class (no subclass)
- `static`: class-level, shared, mutable state pitfalls
- `this`, `super`
- Records (Java 16+): immutable data carriers
- Sealed classes (Java 17+): restrict subclasses for exhaustive pattern matching
- Composition vs inheritance: "has-a" preferred over "is-a"
- Pattern matching (Java 16+): `instanceof` with type variable

**Key Q&A:**
- Interface vs abstract class — when each, Java 8+ context
- Overloading (compile time) vs overriding (runtime) — explain dispatch
- `final` in three contexts — variable, method, class
- Type erasure — what's lost at runtime, what can't you do
- PECS — explain with a method signature example
- Checked vs unchecked exceptions — when to use each
- Composition vs inheritance — give a real refactoring example
- `record` vs regular class — what you get for free, what you don't

**Code patterns:**
- Builder pattern: `Account.builder().name("Alice").balance(1000).build()`
- Sealed + pattern matching: exhaustive `switch` with no default
- Records: compact constructor for validation

**File:** [`v2/03_oop_language.md`](v2/03_oop_language.md)

---

### 04 — Exception Handling
**Priority: 🟡 High**

**Topics covered:**
- Exception hierarchy: `Throwable → Error / Exception → RuntimeException`
- Checked vs unchecked exceptions
- Try-with-resources (AutoCloseable): suppressed exceptions, reverse order close
- Custom exceptions: domain context, unchecked preference
- Exception chaining: preserving root cause
- Multi-catch (Java 7+)
- Best practices: catch specific not broad, throw early catch late, don't swallow
- Spring global error handling: `@RestControllerAdvice` + `@ExceptionHandler`

**Key Q&A:**
- Exception hierarchy — where Error fits, when to catch
- When to create a custom exception — include context
- Try-with-resources — how it works, suppressed exceptions
- `finally` vs try-with-resources
- What happens if you catch `Exception` broadly?

**File:** [`v2/04_exceptions.md`](v2/04_exceptions.md)

---

### 05 — Strings & Immutability
**Priority: 🟡 High**

**Topics covered:**
- String immutability: why it matters (thread safety, security, String pool, hashCode caching)
- String pool (intern pool): since Java 7 it's in the heap
- String comparison: `==` vs `equals()`
- `StringBuilder` vs `StringBuffer`: when to use
- Loop concatenation O(n²) problem
- String methods: `trim/strip`, `substring`, `indexOf`, `contains`, `split`, `toCharArray`
- `char[]` for sensitive data (passwords, PINs)
- How to create an immutable class (5 rules)

**Key Q&A:**
- Why is `String` immutable?
- What's the String pool?
- `==` vs `equals()` for Strings?
- `String` vs `StringBuilder` vs `StringBuffer`?
- Why not concatenate strings in a loop?
- `char[]` vs `String` for passwords?

**File:** [`v2/05_strings.md`](v2/05_strings.md)

---

### 06 — Java 8+ Features
**Priority: 🟡 High**

**Topics covered:**
- Lambdas: anonymous function syntax, effectively final captures
- Functional interfaces: core ones (`Function`, `Predicate`, `Consumer`, `Supplier`)
- Streams: `map` vs `flatMap`, intermediate vs terminal operations
- `Collectors.groupingBy` with downstream collectors
- `Optional`: `of`, `ofNullable`, `orElse` vs `orElseGet`, `map`, `flatMap`, `filter`
- Method references: static, instance on type, instance on object, constructor
- `var` (Java 10) for local variable type inference
- Text blocks (Java 15): multi-line string literals
- Switch expressions (Java 14+): arrow syntax, return values
- `Stream.toList()` (Java 16) vs `Collectors.toList()`

**Key Q&A:**
- `map` vs `flatMap` — give an example with nested collections
- Intermediate vs terminal stream operations
- `orElse` vs `orElseGet` — when the difference matters
- Four types of method references
- `Collectors.groupingBy` with downstream collector
- When streams are a bad choice — give 3 reasons

**File:** [`v2/06_java8_plus.md`](v2/06_java8_plus.md)

---

### 07 — JVM Internals
**Priority: 🟢 Medium**

**Topics covered:**
- Heap vs stack: where what lives
- Generational GC: Young Gen (Eden + Survivor), Old Gen, weak generational hypothesis
- Modern GC algorithms: G1 (default Java 9+), ZGC, Shenandoah, Parallel
- Stop-the-world pause
- Class loading: load → link → initialise, three class loaders
- JIT compilation: C1 (client) → C2 (server), tiered compilation
- `OutOfMemoryError` types: heap space, metaspace, unable to create thread, etc.
- `-Xms` vs `-Xmx`: why set them equal

**Key Q&A:**
- Heap vs stack — what lives where
- Generational GC — why it works, Eden → Survivor → Old Gen
- G1 vs ZGC — when each
- Five types of OutOfMemoryError

**File:** [`v2/07_jvm.md`](v2/07_jvm.md)

---

### 08 — Design Patterns
**Priority: 🟢 Medium**

**Topics covered:**
- Singleton: enum (recommended), double-checked locking, eager init
- Factory Method vs Abstract Factory
- Builder: when you have many parameters, telescoping constructor problem
- Strategy: family of algorithms, lambdas often replace strategy classes
- Observer: one-to-many dependency, event-driven
- Proxy: control access, Spring AOP uses it
- Template Method: skeleton + subclass overrides specific steps
- Decorator: wraps object to add behaviour, Java I/O

**Key Q&A:**
- Singleton — enum vs DCL vs eager init?
- Factory vs Builder — when each
- Strategy — modern Java lambda version
- Proxy — how Spring uses it for AOP
- Decorator — Java I/O example

**File:** [`v2/08_design_patterns.md`](v2/08_design_patterns.md)

---

### 09 — SOLID & Design Principles
**Priority: 🟢 Medium**

**Topics covered:**
- **SRP** (Single Responsibility): one reason to change
- **OCP** (Open/Closed): open for extension, closed for modification
- **LSP** (Liskov Substitution): subtypes substitutable without breaking correctness
- **ISP** (Interface Segregation): many small interfaces > one fat interface
- **DIP** (Dependency Inversion): depend on abstractions, not concretions
- **DRY** (Don't Repeat Yourself): single source of truth
- **KISS** (Keep It Simple): simplicity is key
- **YAGNI** (You Aren't Gonna Need It): don't over-engineer

**Key Q&A:**
- All five SOLID principles — one sentence each
- Give a violation example for LSP
- DIP — how Spring DI implements it
- When DRY goes too far (premature abstraction)

**File:** [`v2/09_solid.md`](v2/09_solid.md)

---

### 10 — Testing
**Priority: 🟢 Medium**

**Topics covered:**
- JUnit 5: `@Test`, lifecycle (`@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`)
- Assertions: `assertEquals`, `assertNotNull`, `assertThrows`, `assertAll`
- Mockito: `@Mock`, `@InjectMocks`, `when/thenReturn`, `verify`
- `mock` vs `spy`
- Test doubles: dummy, stub, mock, fake, spy
- Unit vs integration vs E2E testing
- Testing pyramid: many unit, fewer integration, fewest E2E
- How to test a private method: you don't
- TDD: Red → Green → Refactor

**Key Q&A:**
- JUnit 5 lifecycle annotations — `@BeforeEach` vs `@BeforeAll`
- Mockito — `when/thenReturn`, `verify`, `@Mock` vs `@Spy`
- Testing pyramid — unit vs integration vs E2E
- How to test code that throws exceptions

**File:** [`v2/10_testing.md`](v2/10_testing.md)

---

## 🟡 Spring Ecosystem

These topics appear in **~30% of Karat questions**. Essential for Spring Boot production code.

### 11 — Spring Core & Dependency Injection
**Priority: 🟡 High**

**Topics covered:**
- IoC (Inversion of Control): framework controls object creation & wiring
- Three DI types: constructor (preferred), setter, field
- Why constructor injection is best: explicit dependencies, immutability, testable
- Bean scopes: singleton, prototype, request, session, application
- Prototype into singleton problem: three solutions (`@Lookup`, `ObjectFactory<T>`, `Provider<T>`)
- `@Qualifier` for disambiguation
- Bean lifecycle: construction → `@PostConstruct` → ready → `@PreDestroy`
- Spring profiles: environment-specific bean activation, `application-{profile}.yml`
- `@ConfigurationProperties` vs `@Value`: structured config type safety
- Circular dependency resolution & constructor injection failure

**Key Q&A:**
- Three DI types — constructor (preferred), setter, field
- Why constructor injection is preferred
- Bean scopes — singleton, prototype, request, session
- Prototype into singleton problem — three solutions
- Bean lifecycle — `@PostConstruct` / `@PreDestroy`
- Circular dependency — how Spring resolves it, why constructor injection fails

**File:** [`v2/11_spring_core.md`](v2/11_spring_core.md)

---

### 12 — Spring Boot
**Priority: 🟡 High**

**Topics covered:**
- Spring Boot vs Spring Framework: opinionated auto-configuration, embedded server
- Auto-configuration: `@EnableAutoConfiguration`, `@Conditional*` annotations
- Starters: curated dependency bundles (`spring-boot-starter-web`, etc.)
- `@Transactional`: proxy mechanism, self-invocation trap, rollback rules
- `@Async`: separate thread, caveats (proxy, executor, exception handling)
- Spring Boot Actuator: `/health`, `/metrics`, `/info`, custom health indicators
- Global error handling: `@RestControllerAdvice` + `@ExceptionHandler`
- Scheduling: `@EnableScheduling`, `@Scheduled`, `ThreadPoolTaskScheduler`
- `application.yml` property precedence
- Test slices: `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`

**Key Q&A:**
- What is Spring Boot and how is it different from Spring?
- How does auto-configuration work?
- What are starters?
- `@Transactional` — proxy mechanism, self-invocation trap, rollback rules
- `@Async` — three caveats (proxy, executor, exception handling)
- Actuator endpoints — `/health`, `/metrics`
- Property precedence — command line > env > profile yml > base yml

**File:** [`v2/12_spring_boot.md`](v2/12_spring_boot.md)

---

### 13 — Spring REST & Web
**Priority: 🟡 High**

**Topics covered:**
- `@Controller` vs `@RestController`
- Request mapping: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
- Path variables, query params, request body
- Idempotent HTTP methods: GET, PUT, DELETE vs POST
- JSON serialisation/deserialisation: Jackson, `@JsonProperty`, `@JsonIgnore`
- Request validation: `@Valid`, Bean Validation annotations
- Global exception handling: `@RestControllerAdvice`, custom error response
- `ResponseEntity`: return full HTTP response
- API versioning: URI, header, query param approaches
- `RestTemplate` vs `WebClient` vs `RestClient` (Spring 6.1)
- Spring Security basics: `SecurityFilterChain`, JWT, OAuth2
- CORS: cross-origin resource sharing configuration
- Content negotiation: JSON/XML support

**Key Q&A:**
- `@RestController` — what it combines
- Idempotent HTTP methods — which ones and why it matters
- Request validation — `@Valid` + Bean Validation annotations
- Global exception handling — `@RestControllerAdvice` + `@ExceptionHandler`
- `RestTemplate` vs `WebClient` vs `RestClient` — when each

**File:** [`v2/13_spring_rest.md`](v2/13_spring_rest.md)

---

### 14 — Spring AOP & Data
**Priority: 🟢 Medium**

**Topics covered:**

**AOP:**
- What AOP solves: cross-cutting concerns (logging, security, caching, transactions)
- Key terms: aspect, joinpoint, pointcut, advice, weaving
- Advice types: `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`
- Spring AOP internals: proxy-based, JDK dynamic proxy vs CGLIB
- Pointcut expressions: examples
- Self-invocation limitation: `this.method()` bypasses proxy

**Spring Data / JPA:**
- JPA (specification) vs Hibernate (implementation) vs Spring Data JPA
- N+1 problem: lazy-loaded relationships, solutions
- Entity lifecycle: transient, managed, detached, removed
- Optimistic vs pessimistic locking: `@Version`, `@Lock`
- Spring Data query methods: method names → queries, `@Query`
- Transaction propagation: REQUIRED, REQUIRES_NEW, SUPPORTS, MANDATORY, etc.
- Spring caching: `@Cacheable`, `@CacheEvict`, `@CachePut`

**Key Q&A:**
- AOP advice types — Before, After, AfterReturning, AfterThrowing, Around
- Spring AOP proxy mechanism — why self-invocation doesn't work
- N+1 problem — what it is, three fixes (JOIN FETCH, @EntityGraph, EAGER)
- Optimistic vs pessimistic locking — when each
- Transaction propagation — REQUIRED vs REQUIRES_NEW
- Spring caching — `@Cacheable`, `@CacheEvict`

**File:** [`v2/14_spring_aop_data.md`](v2/14_spring_aop_data.md)

---

## 🟠 Specialized Topics

### 15 — Kafka Fundamentals
**Priority: 🟡 High**

**Topics covered:**
- Kafka: distributed event streaming platform, immutable append-only log
- Architecture: broker, topic, partition, producer, consumer, consumer group
- Partitions: parallelism, ordering (per-partition only), scalability
- Partition key: determines target partition, guarantees ordering for that key
- Consumer groups: partition assignment, multiple independent groups (fan-out)
- Rebalancing: when it happens, impact, minimisation strategies
- Exactly-once semantics: idempotent producer, transactional API, read_committed
- At-least-once vs at-most-once: offset commit timing
- Dead Letter Topic (DLT): poison pill handling
- Offset management: `auto.commit=true` vs manual
- Replication and ISR (In-Sync Replicas): durability guarantees
- Log compaction vs retention
- Spring Kafka: `KafkaTemplate`, `@KafkaListener`, error handling

**Key Q&A:**
- Kafka vs traditional MQ — append-only log, consumer groups, replay
- Partitions — parallelism, ordering guarantee (per-partition only)
- Partition key — how it determines ordering
- Consumer group rebalancing — when it happens, how to minimise
- Exactly-once — idempotent producer + transactional API + read_committed
- At-least-once vs at-most-once — offset commit timing
- Dead Letter Topic — when and how
- ISR + `acks=all` + `min.insync.replicas` — durability guarantee

**File:** [`v2/15_kafka.md`](v2/15_kafka.md)

---

## 🔴 Debugging & Code Reading

These topics appear in **~60% of Karat questions** (often combined with functional tasks).

### 16 — Bug Taxonomy & Debugging
**Priority: 🔴 Critical**

**Topics covered:**
- Bug types: off-by-one, null handling, wrong operator, mutation during iteration
- Integer overflow, resource leak, wrong base case, check-then-act (race condition)
- Visibility issues (concurrency), wrong initialisation, floating-point
- The dry-run discipline: write columns, read aloud, update state, trace branches
- Five classic concurrency bugs: singleton race, volatile counter, check-then-act, etc.

**Key Q&A:**
- Name the 11 common bug types
- Walk through the dry-run discipline
- Singleton race condition: what can go wrong
- Off-by-one in loops, substring, array access

**File:** [`v2/16_debugging.md`](v2/16_debugging.md)

---

### 17 — Debugging Problems
**Priority: 🔴 Practice**

**20 predict-the-output & find-the-bug problems:**
- Static list accumulator, integer cache, string pool, autoboxing NPE
- String concatenation with operators, format exceptions, concurrent modification
- Lambda closure capture, try-with-resources order, finally overrides
- Switch fall-through, integer division, floating-point equality, shallow copy
- Arrays.asList(), method hiding vs overriding, String.split()
- Stream reduce, final keyword, volatile counter race

**Companion file:** [`v2/17_debugging_problems.md`](v2/17_debugging_problems.md)

---

## 📊 Data Structures Deep-Dive

### DataStructures.md
**Complete reference:** Arrays, ArrayList, LinkedList, HashMap, TreeMap, HashSet, TreeSet, Queue/

Deque, PriorityQueue, Stack, Graphs, trees, heaps.

Each section covers: etymology, real-world example, problem it solves, complexity table, method cheat sheet, internals, gotchas, pattern template, DSA problem types.

**File:** [`v2/DataStructures/DataStructures.md`](v2/DataStructures/DataStructures.md)

---

## 🎯 Practice & Companion Drill Sets

### Debugging Problems
**20 predict-the-output & find-the-bug problems** (25–30 sec each in your head).

**File:** [`v2/Practice/karat-debugging-problems.md`](v2/Practice/karat-debugging-problems.md)

---

### Ladder Problems
**10 OOP debug-and-extend exercises** (35 min each with test suite).

Patterns: missing filters, wrong aggregations, shallow copy bugs, concurrent modification, arithmetic edge cases.

**File:** [`v2/Practice/karat-ladder-problems.md`](v2/Practice/karat-ladder-problems.md)

---

### Complete Solutions
**Full worked solutions** for all debugging (DBG-01–23) and ladder problems (LADDER-01–10).

**File:** [`v2/Practice/karat-solutions.md`](v2/Practice/karat-solutions.md)

---

### Practice Index
**Master index** with problem mapping to real Karat question bank patterns.

**File:** [`v2/Practice/karat-practice-INDEX.md`](v2/Practice/karat-practice-INDEX.md)

---

## 📖 How to Study — Recommended Discipline

### Study Mode (Per Topic — 60 minutes)

1. **Read the 📚 Study Material section** (20 min)
   - Focus on mechanism explanations, not just definitions
   - Trace code examples with a debugger mentally

2. **Answer the Rapid-Fire Q&A aloud** (20 min)
   - ≤30 seconds per answer
   - Use the explanation below each Q as a crutch if you stumble
   - Record yourself — listen back for clarity

3. **Code Pattern Review** (10 min)
   - Understand why each pattern is there
   - Could you write it from memory?

4. **Self-Check Checklist** (10 min)
   - Can you answer each item cold?
   - If not, re-read the material for that item

### Practice Mode (Debugging Problems — 30 minutes)

1. **Predict the output** (10 sec per problem)
   - Write it on paper in the margin
   - Reason aloud about why

2. **Name the mechanism or exception** (10 sec)
   - **Not just ** "it crashes" — name the class: `ConcurrentModificationException`
   - Understand the root cause

3. **Check the solution** (10 min)
   - Review [`v2/Practice/karat-solutions.md`](v2/Practice/karat-solutions.md)
   - If you were wrong, **trace why** and re-read the relevant topic

### Practice Mode (Ladder Problems — 40 minutes per problem)

1. **Read all three classes** — top to bottom (5 min)
   - What's the bug? What's the extension?

2. **Dry-run the failing test** (10 min)
   - Column headers for all variables that matter
   - Read each line aloud
   - Where does the test fail?

3. **Fix the bug** (10 min)
   - Confirm the fix with the test suite

4. **Implement extension** (10 min)
   - Write from memory if possible

5. **Refactor & review** (5 min)
   - Is there a simpler implementation?

---

## 🚀 Interview Day Preparation

### Final Review (Day Before)
- ✅ Re-read the syllabus: [`v2/00_INDEX.md`](v2/00_INDEX.md)
- ✅ Drill 5 random debugging problems in your head (15 min)
- ✅ Trace one ladder problem start-to-finish (35 min)
- ✅ Sleep well

### Day Of
- ✅ Physically write pseudo-code on paper (or whiteboard) — don't just think
- ✅ **Define the problem in your own words first**
- ✅ **Trace through your solution** with the test data hand-in-hand
- ✅ **Name the mechanisms and exceptions** by their exact class name
- ✅ **Speak aloud the entire time** — even if just to yourself

---

## 📋 Index of All Topics

| # | Topic | Priority | File |
|---|-------|:--------:|------|
| **Core Java** | | | |
| 01 | Concurrency & Threading | 🔴 Critical | [`01_concurrency.md`](v2/01_concurrency.md) |
| 02 | Collections Internals | 🔴 Critical | [`02_collections.md`](v2/02_collections.md) |
| 03 | OOP & Language | 🔴 Critical | [`03_oop_language.md`](v2/03_oop_language.md) |
| 04 | Exception Handling | 🟡 High | [`04_exceptions.md`](v2/04_exceptions.md) |
| 05 | Strings & Immutability | 🟡 High | [`05_strings.md`](v2/05_strings.md) |
| 06 | Java 8+ Features | 🟡 High | [`06_java8_plus.md`](v2/06_java8_plus.md) |
| 07 | JVM Internals | 🟢 Medium | [`07_jvm.md`](v2/07_jvm.md) |
| 08 | Design Patterns | 🟢 Medium | [`08_design_patterns.md`](v2/08_design_patterns.md) |
| 09 | SOLID & Design Principles | 🟢 Medium | [`09_solid.md`](v2/09_solid.md) |
| 10 | Testing | 🟢 Medium | [`10_testing.md`](v2/10_testing.md) |
| **Spring Ecosystem** | | | |
| 11 | Spring Core & DI | 🟡 High | [`11_spring_core.md`](v2/11_spring_core.md) |
| 12 | Spring Boot | 🟡 High | [`12_spring_boot.md`](v2/12_spring_boot.md) |
| 13 | Spring REST & Web | 🟡 High | [`13_spring_rest.md`](v2/13_spring_rest.md) |
| 14 | Spring AOP & Data | 🟢 Medium | [`14_spring_aop_data.md`](v2/14_spring_aop_data.md) |
| **Kafka** | | | |
| 15 | Kafka Fundamentals | 🟡 High | [`15_kafka.md`](v2/15_kafka.md) |
| **Debugging** | | | |
| 16 | Bug Taxonomy & Debugging | 🔴 Critical | [`16_debugging.md`](v2/16_debugging.md) |
| 17 | Debugging Problems (20) | 🔴 Practice | [`17_debugging_problems.md`](v2/17_debugging_problems.md) |
| **Supporting** | | | |
| — | Data Structures Deep-Dive | 📊 Reference | [`DataStructures/DataStructures.md`](v2/DataStructures/DataStructures.md) |
| — | Practice Debugging (DBG-01–23) | 🐛 Practice | [`Practice/karat-debugging-problems.md`](v2/Practice/karat-debugging-problems.md) |
| — | Practice Ladder (LADDER-01–10) | 🪜 Practice | [`Practice/karat-ladder-problems.md`](v2/Practice/karat-ladder-problems.md) |
| — | Complete Solutions | ✅ Reference | [`Practice/karat-solutions.md`](v2/Practice/karat-solutions.md) |

---

## 🎓 Philosophy

This guide is built on three principles:

1. **Mechanism over memorisation.** You don't memorise answers. You understand *why* a mechanism works, *why* a bug occurs, and what class/method name to use. Then you can derive the answer in the interview.

2. **Rapid-fire before deep-dive.** Every topic starts with 30-second Q&A drills. Only if you stumble do you read the explanation. This trains your brain to recognise patterns fast.

3. **Dry-run discipline is non-negotiable.** The single most powerful debugging skill is the ability to trace code by hand. This guide drills it obsessively.

---

## 📞 Troubleshooting

**Q: I'm stuck on a topic.**
- A: Re-read the study material, then do the Q&A aloud. If still stuck, pick a different topic and come back.

**Q: I got all the debugging problems wrong.**
- A: This is normal. Flip to the solutions, trace why you went wrong, re-read the relevant topic. Tomorrow, pick 5 random ones and try again.

**Q: I don't have time.**
- A: Do Path 3 (Critical Only, 1 week) or Path 4 (Emergency Mode, 3 days). Focus on [01](#01--concurrency--threading), [02](#02--collections-internals), [03](#03--oop--language-fundamentals), [16](#16--bug-taxonomy--debugging-critical). Practice, practice, practice.

**Q: Should I memorise the Q&A?**
- A: No. Understand the mechanism. Then you'll naturally answer the Q in the interview.

---

## ✅ Final Checklist Before Interview

- [ ] Can I explain `synchronized` without hesitation?
- [ ] Can I trace a HashMap collision scenario?
- [ ] Can I spot an off-by-one error in 5 seconds?
- [ ] Can I name the exact exception for `remove()` during iteration?
- [ ] Can I explain PECS with a code example?
- [ ] Can I trace a deadlock scenario and name all four conditions?
- [ ] Can I write a CompletableFuture pipeline from memory?
- [ ] Can I explain the N+1 problem and three fixes?
- [ ] Can I dry-run a recursive function and spot the base case?
- [ ] Can I explain why `@Transactional` self-invocation doesn't work?

If you checked all ✅, you're ready.

---

## 📄 Version & Maintenance

**Version:** KaratPrepV2 — Optimised for Java 11+, Spring 5+, Kafka 3+

**Last updated:** April 2026

**Maintained by:** Karat Coaching Team

---

## 📞 Getting Help

- **Within the docs:** Every topic has a "Can you answer these cold?" checklist at the end. Use that.
- **Practice:** All debugging and ladder problems have full solutions in `Practice/karat-solutions.md`.
- **Feedback:** Re-read this README if you're unsure where to start.

---

Good luck. You've got this. 🚀

