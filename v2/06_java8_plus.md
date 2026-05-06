# 06 — Java 8+ Features

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/06_java8_plus_doubts.md) · [← Prev: 05 Strings](./05_strings.md) · [Next: 07 JVM →](./07_jvm.md)
>
> **Priority:** 🟡 High · **Related topics:** [02 Collections (Streams)](./02_collections.md) · [03 OOP (functional interfaces, records)](./03_oop_language.md) · [01 Concurrency (CompletableFuture)](./01_concurrency.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — why Java needed an upgrade in 2014
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — a stream pipeline, lazy
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — lambda + stream + Optional
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Lambdas](#41-lambdas--inline-functions)
   - [4.2 Functional interface taxonomy](#42-functional-interface-taxonomy)
   - [4.3 Method references](#43-method-references-four-flavours)
   - [4.4 Streams pipeline](#44-streams--pipeline-of-lazy-stages)
   - [4.5 Collectors](#45-collectors-the-ones-you-actually-use)
   - [4.6 flatMap](#46-flatmap--flatten-nested-structures)
   - [4.7 Optional](#47-optional--safe-handling-of-might-be-missing)
   - [4.8 var (Java 10+)](#48-var-java-10--local-variable-type-inference)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 Laziness and short-circuiting](#51-stream-laziness-and-short-circuiting)
   - [5.2 Parallel streams](#52-parallel-streams--and-when-not-to-use-them)
   - [5.3 Switch expressions](#53-switch-expressions-java-14)
   - [5.4 Records (Java 16+)](#54-records-java-16--cross-link-to-03)
   - [5.5 Sealed classes (Java 17+)](#55-sealed-classes-java-17--cross-link-to-03)
   - [5.6 Pattern matching](#56-pattern-matching)
   - [5.7 Sequenced collections (Java 21+)](#57-sequenced-collections-java-21)
   - [5.8 Virtual threads (Java 21+)](#58-virtual-threads-java-21--cross-link-to-01)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Pre-2014, Java had a verbosity problem.

To sort a list of names by length, you wrote eight lines of boilerplate — a brand-new anonymous inner class implementing `Comparator`, with `@Override` and `public int compare(...)`. You wrote it because Java required it, not because the *intent* was complicated.

To filter a list, transform it, and collect it, you wrote three nested for-loops with three intermediate `ArrayList`s. The actual logic — "keep the active users, get their emails, into one list" — got buried under syntax.

To represent "the user might not exist", you returned `null`. Every caller had to remember to check for null. They forgot. Your logs filled with `NullPointerException`.

Java 8 (2014) is the language's hour-long course-correction.

- A **lambda** is a tiny on-the-spot function — the comparator, but in one line. It's a sticky note with instructions you hand to a method.
- A **stream** is an assembly line for a collection: items flow in one end, get filtered, transformed, collected at the other. Each station does one thing; the pipeline reads top to bottom.
- An **`Optional`** is an envelope that explicitly says "there may or may not be a value inside". Open it deliberately. The compiler reminds you. NPEs disappear.
- A **functional interface** is the *type* a lambda fills in for: any interface with one method. Java has built-in ones (`Function`, `Predicate`, `Consumer`, `Supplier`) for the four shapes that cover 90% of cases.

> Java 9–21 then keeps polishing: `var`, switch expressions, text blocks, records, sealed classes, pattern matching, virtual threads. Each one removes a category of boilerplate that Java 8 didn't get to.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

You have a list of `Order` objects. You want: from active orders, the customer email, deduplicated, top 10.

In a stream pipeline, that reads top to bottom:

```
orders                          // SOURCE          — Stream<Order>
    .filter(o -> o.active)      // INTERMEDIATE    — keeps active, lazy
    .map(o -> o.email)          // INTERMEDIATE    — turns each into email, lazy
    .distinct()                 // INTERMEDIATE    — removes dupes, stateful
    .limit(10)                  // INTERMEDIATE    — caps at 10, short-circuiting
    .toList()                   // TERMINAL        — triggers execution, returns List<String>
```

What's quietly profound here: **nothing in the pipeline runs until `.toList()`**. The intermediate stages just describe the work. When `.toList()` fires, the JVM runs the pipeline **one element at a time**:

| Element | filter | map | distinct | limit | collected? |
|---|---|---|---|---|---|
| order #1 (active) | passes | "alice@" | new | count=1 | ✅ |
| order #2 (inactive) | drops | — | — | — | ❌ |
| order #3 (active) | passes | "bob@" | new | count=2 | ✅ |
| order #4 (active) | passes | "alice@" | duplicate — drops | — | ❌ |
| … | … | … | … | … | … |
| order #N | … | "x@" | new | count=10 | ✅ |
| order #N+1 (active) | passes | "y@" | new | count would be 11 — `limit(10)` says STOP | terminate |

That short-circuit is why streaming a million orders to get 10 emails doesn't process all million — `limit` cuts off the pipeline as soon as enough survive.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class Java8Demo {
    record Order(String email, boolean active) {}      // record (Java 16+) — see §5.4

    public static void main(String[] args) {
        List<Order> orders = List.of(
            new Order("alice@x.com", true),
            new Order("bob@x.com",   false),
            new Order("alice@x.com", true),
            new Order("carol@x.com", true)
        );

        // Lambda — Predicate<Order> stored as a value
        Predicate<Order> isActive = o -> o.active();

        // Stream pipeline — lazy until .toList()
        List<String> emails = orders.stream()
            .filter(isActive)                 // keep active
            .map(Order::email)                // method reference — same as o -> o.email()
            .distinct()                       // dedupe
            .toList();                        // Java 16+ — unmodifiable
        System.out.println(emails);            // [alice@x.com, carol@x.com]

        // Optional — safe lookup
        Optional<Order> first = orders.stream().filter(isActive).findFirst();
        String firstEmail = first
            .map(Order::email)                // transform inside the envelope
            .orElse("none");                   // unwrap with default
        System.out.println(firstEmail);
    }
}
```

What just happened: a lambda became a typed value (`Predicate<Order>`). A stream pipeline filtered, mapped, and deduped without a single explicit loop or temporary list. An `Optional` wrapped a "maybe missing" result and let us provide a default in one expression — no `if (x != null)` ladder.

---

## 4. Build Up — Practical Patterns

### 4.1 Lambdas — inline functions

A lambda is a value of a **functional interface** (an interface with exactly one abstract method). It's the modern replacement for `new SomeInterface() { @Override … }`.

```java
// Anonymous class (pre-Java 8)
Comparator<String> byLen = new Comparator<String>() {
    @Override public int compare(String a, String b) { return a.length() - b.length(); }
};

// Lambda — same thing
Comparator<String> byLen2 = (a, b) -> a.length() - b.length();

// Syntax variants
x -> x * 2                                // single param, parens optional
(x, y) -> x + y                            // multiple params
() -> System.out.println("hello")         // no params
(String x) -> x.length()                   // explicit type (usually inferred)
x -> { var y = x * 2; return y; }          // block body — needs return
```

**Effectively-final capture.** A lambda may use local variables only if they are *effectively final* (assigned once, never reassigned). The JVM captures the *value* at creation, not a reference, so concurrent mutation isn't a worry.

```java
int factor = 3;                            // effectively final
Function<Integer, Integer> mul = x -> x * factor;   // captures the VALUE 3
// factor = 4;  // ❌ would invalidate the capture — won't compile
```

### 4.2 Functional interface taxonomy

| Interface | Signature | Use | Example |
|-----------|-----------|-----|---------|
| `Function<T,R>` | `T → R` | Transform | `user -> user.getName()` |
| `Predicate<T>` | `T → boolean` | Filter / test | `user -> user.isActive()` |
| `Consumer<T>` | `T → void` | Side effect | `user -> log.info(user)` |
| `Supplier<T>` | `() → T` | Factory / lazy default | `() -> new ArrayList<>()` |
| `UnaryOperator<T>` | `T → T` | Same-type transform | `s -> s.toUpperCase()` |
| `BiFunction<T,U,R>` | `(T,U) → R` | Two-arg transform | `(a,b) -> a + b` |
| `BinaryOperator<T>` | `(T,T) → T` | Two same-type → one | `Integer::sum` |

```java
// Composing functions
Function<String, String> trim  = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> pipe  = trim.andThen(upper);
pipe.apply("  hello  ");                       // "HELLO"

// Composing predicates
Predicate<Employee> senior   = e -> e.years()   > 5;
Predicate<Employee> highPaid = e -> e.salary() > 100_000;
Predicate<Employee> seniorAndHighPaid = senior.and(highPaid);
```

### 4.3 Method references — four flavours

Compact lambda syntax when the body just calls another method. Four kinds:

```java
// 1. STATIC method
Function<String, Integer> parse = Integer::parseInt;        //  s -> Integer.parseInt(s)

// 2. INSTANCE method bound to a type — receiver becomes the first arg
Function<String, String> upper = String::toUpperCase;        //  s -> s.toUpperCase()

// 3. INSTANCE method bound to a SPECIFIC object
String prefix = "Hello, ";
Function<String, String> greet = prefix::concat;             //  s -> prefix.concat(s)

// 4. CONSTRUCTOR
Supplier<List<String>> factory = ArrayList::new;             //  () -> new ArrayList<>()
Function<String, File>  fileFactory = File::new;             //  s -> new File(s)
```

### 4.4 Streams — pipeline of lazy stages

```java
List<String> names = employees.stream()                      // SOURCE
    .filter(e -> e.salary() > 50_000)                        // intermediate, stateless, lazy
    .map(Employee::name)                                     // intermediate, stateless, lazy
    .sorted()                                                // intermediate, STATEFUL — buffers
    .limit(10)                                               // intermediate, short-circuiting
    .toList();                                                // TERMINAL — triggers execution
```

| Operation kind | Examples | Notes |
|---|---|---|
| Stateless intermediate | `filter`, `map`, `flatMap`, `peek` | One element at a time |
| Stateful intermediate | `sorted`, `distinct`, `skip` | Must buffer or remember |
| Short-circuiting intermediate | `limit`, `takeWhile` (Java 9+) | Can stop mid-stream |
| Terminal | `toList()`, `collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch` | Triggers execution |
| Short-circuiting terminal | `findFirst`, `findAny`, `anyMatch`, `allMatch`, `noneMatch` | May stop early |

Streams are **single-use**. `s.forEach(...)` after another terminal call → `IllegalStateException`. Reuse means `.stream()` again on the source.

### 4.5 Collectors (the ones you actually use)

```java
// Group by → Map<K, List<V>>
Map<String, List<Employee>> byDept =
    employees.stream().collect(Collectors.groupingBy(Employee::dept));

// Group by + downstream aggregator
Map<String, Long> countByDept =
    employees.stream().collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));

Map<String, Double> avgSalaryByDept =
    employees.stream().collect(Collectors.groupingBy(Employee::dept,
        Collectors.averagingDouble(Employee::salary)));

// Partition by predicate → Map<Boolean, List<…>>
Map<Boolean, List<Employee>> split =
    employees.stream().collect(Collectors.partitioningBy(e -> e.salary() > 100_000));

// Join into a string
String csv = employees.stream().map(Employee::name)
    .collect(Collectors.joining(", ", "[", "]"));

// Build a Map (with merge function for duplicate keys)
Map<String, Employee> byId =
    employees.stream().collect(Collectors.toMap(Employee::id, Function.identity(),
        (existing, replacement) -> existing));
```

### 4.6 flatMap — flatten nested structures

```java
// orders.stream().map(Order::items)  → Stream<List<Item>>   ← nested
// orders.stream().flatMap(o -> o.items().stream()) → Stream<Item>   ← flat

List<Item> allItems = orders.stream()
    .flatMap(o -> o.items().stream())
    .distinct()
    .toList();
```

> `map`: 1 → 1. `flatMap`: 1 → N (then concatenate).

### 4.7 Optional — safe handling of "might be missing"

```java
Optional<User> user = repo.findById(id);     // never returns null

String email = user
    .map(User::profile)                       // Optional<Profile>  — only runs if user present
    .map(Profile::email)                      // Optional<String>
    .orElse("unknown@x.com");                  // unwrap with default
```

**Choosing your unwrap:**

```java
opt.orElse(expensive());                      // ❌ expensive() runs ALWAYS, even when present
opt.orElseGet(() -> expensive());             // ✅ expensive() only runs when EMPTY
opt.orElseThrow(() -> new NotFound(id));      // throw on missing
opt.ifPresent(u -> notify(u));                 // act if present
opt.ifPresentOrElse(u -> notify(u),
                    () -> log.warn("missing")); // Java 9+
```

**Anti-patterns to avoid:**

```java
if (opt.isPresent()) return opt.get();        // ❌ defeats the type — use orElse/orElseGet
return opt.orElse(default);                   // ✅

class User { Optional<String> middleName; }   // ❌ never on a field
void process(Optional<String> arg) {}         // ❌ never as a parameter
return Optional.of(emptyList());              // ❌ return an empty collection instead
```

### 4.8 `var` (Java 10+) — local variable type inference

```java
var list = new ArrayList<String>();           // inferred ArrayList<String>
var stream = list.stream();                   // inferred Stream<String>
var users = repo.findAll();                   // inferred from method return

// Allowed only on LOCAL variables. Not fields, not parameters, not return types.
// Don't use when the type adds genuine information:
//   var x = service.process();                // ❌ what's x? Reader can't tell.
```

---

## 5. Going Deep — Interview-Level Material

### 5.1 Stream laziness and short-circuiting

Intermediate stages are pure description. The pipeline doesn't run until a terminal operation pulls items through. Two consequences worth being able to explain:

1. **Element-at-a-time.** A `filter`-then-`map`-then-`limit(3)` over a million items processes maybe ~5 — not all million through filter, then all through map. Items flow individually; `limit` short-circuits the upstream.
2. **Side effects in stateless ops are dangerous.** `peek(System.out::println)` may print 0 items, or all items, depending on whether the terminal short-circuits and on JIT decisions. Don't rely on `peek` for anything but debugging.

```java
Stream.of(1, 2, 3, 4, 5)
    .peek(x -> System.out.println("filter: " + x))
    .filter(x -> x % 2 == 0)
    .findFirst();
// Prints "filter: 1", "filter: 2", returns Optional.of(2). Doesn't process 3, 4, 5.
```

### 5.2 Parallel streams — and when not to use them

```java
list.parallelStream().map(this::heavyCompute).toList();
// or:
list.stream().parallel().toList();
```

Backed by `ForkJoinPool.commonPool()` — same pool as `CompletableFuture`'s default executor. Three caveats:

1. **It's the *common* pool.** Blocking work in a parallel stream starves other things using the common pool.
2. **Order may matter.** `forEach` on a parallel stream gives no order guarantee. `forEachOrdered` does, at a cost.
3. **Source matters.** `ArrayList` and arrays parallelise efficiently. `LinkedList`, `Stream.iterate`, `BufferedReader.lines()` do not — splitting them is O(n).

Rule of thumb: parallel streams help only on CPU-bound, large, splittable, stateless work. Otherwise the overhead exceeds the benefit.

### 5.3 Switch expressions (Java 14+)

`switch` is now an expression that returns a value, with arrow syntax (no fall-through) and `yield` inside blocks:

```java
String label = switch (status) {
    case ACTIVE   -> "Active";
    case INACTIVE -> "Inactive";
    case PENDING  -> {
        log.info("pending case");
        yield "Pending";                       // yield = return-from-block
    }
};
```

Switch expressions must be **exhaustive** — cover every case (or include `default`). Combined with sealed types (§5.5), the compiler can verify this without a default.

### 5.4 Records (Java 16+) — cross-link to [03](./03_oop_language.md#53-records-java-16)

Immutable data carriers with auto-generated `equals`/`hashCode`/`toString` and accessors. See [03 §5.3](./03_oop_language.md#53-records-java-16) for full coverage.

```java
public record OrderId(String value) {}
```

### 5.5 Sealed classes (Java 17+) — cross-link to [03](./03_oop_language.md#54-sealed-classes-java-17)

Restrict who can extend/implement. Combine with switch expressions and pattern matching for exhaustive type dispatch. See [03 §5.4](./03_oop_language.md#54-sealed-classes-java-17).

### 5.6 Pattern matching

```java
// Java 16+: pattern matching for instanceof
if (obj instanceof String s) use(s.length());

// Java 21: pattern matching for switch
String describe(Object o) { return switch (o) {
    case Integer i when i > 0 -> "positive: " + i;
    case Integer i            -> "non-positive: " + i;
    case String s             -> "string of length " + s.length();
    case null                 -> "null";
    default                   -> "other";
};}
```

Combined with sealed hierarchies, pattern matching makes "what kind of thing is this?" code dramatically cleaner. See [03 §5.5](./03_oop_language.md#55-pattern-matching-java-16-instanceof-java-21-switch).

### 5.7 Sequenced collections (Java 21+)

A unified API for "ordered" collections. `List`, `Deque`, and `LinkedHashMap` all gain consistent `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `reversed()`. Closes a long-standing inconsistency where you needed `list.get(list.size()-1)` for a `List` but `deque.getLast()` for a `Deque`.

```java
SequencedCollection<String> seq = new ArrayList<>();
seq.addFirst("a"); seq.addLast("z");
seq.getFirst();   // "a"
seq.reversed();   // ["z", "a"]
```

### 5.8 Virtual threads (Java 21+) — cross-link to [01](./01_concurrency.md#510-virtual-threads-java-21)

JVM-scheduled threads that are cheap (KB) instead of expensive (MB). Block-style code stays simple. See [01 §5.10](./01_concurrency.md#510-virtual-threads-java-21).

---

## 6. Memory Aids

### Decision tree: "should I use a stream?"

```
Are you transforming a collection (filter / map / collect)?
├── Yes — use a stream. Pipeline reads top-to-bottom.
└── No
    ├── Need an explicit index, break, or continue?
    │   └── Use a for-loop. Streams force you to fight the API.
    ├── Side effects only (logging, sending events)?
    │   └── for-each loop. Don't use streams for side effects.
    └── Performance critical, tight inner loop?
        └── for-loop. Streams have constant overhead per element.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "What's a lambda?" | A value of a functional interface | "It captures effectively-final locals by value." |
| "Stream `map` vs `flatMap`?" | 1→1 vs 1→N | "flatMap unwraps the inner streams." |
| "`orElse` vs `orElseGet`?" | Eager vs lazy default | "`orElse(expensive())` always runs `expensive()`. `orElseGet(supplier)` only when empty." |
| "When is a parallel stream worth it?" | CPU-bound + large + splittable + stateless | "Otherwise serial is faster." |
| "Why are streams lazy?" | Element-at-a-time + short-circuiting | "A million-item source with `limit(10)` processes ~10 items." |
| "Why is `Optional` for fields bad?" | Serialisation, JPA, framework support | "Use `@Nullable` or empty collection." |
| "What's `var`?" | Local-only inference | "Doesn't change the type system. Just less typing for obvious cases." |

### Three anchor principles

1. **Lambda = a value with a functional-interface type.** Pass it like data.
2. **Stream pipelines are lazy.** Nothing runs until a terminal op.
3. **`Optional` makes "missing" part of the type.** The compiler reminds you to handle it.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: What are lambdas and when do you use them?
**A:** Anonymous-function syntax: `(params) -> expression`. Used wherever a functional interface is expected. Cleaner than anonymous classes. Captured locals must be effectively final.

### Q2: What's a functional interface?
**A:** Interface with exactly one abstract method. Optionally annotated `@FunctionalInterface`. Core ones: `Function<T,R>`, `Predicate<T>`, `Consumer<T>`, `Supplier<T>`, `BiFunction<T,U,R>`, `UnaryOperator<T>`.

### Q3: Streams — `map` vs `flatMap`?
**A:** `map` is one-to-one (`Stream<A>` → `Stream<B>`). `flatMap` is one-to-many followed by concatenation (`Stream<A>` → `Stream<B>` where each `A` produces a stream of `B`s and they're flattened).

### Q4: Stream pipeline — intermediate vs terminal ops?
**A:** Intermediate (lazy): `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `peek`. Terminal (triggers execution): `collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`, `toList()`. Streams are single-use.

### Q5: `Collectors.groupingBy` — how does it work?
**A:** Groups stream elements by a classifier, returning `Map<K, List<V>>`. With a downstream collector: `groupingBy(Employee::dept, counting())` returns `Map<String, Long>`.

### Q6: When are streams a bad choice?
**A:** Performance-critical inner loops (constant overhead per element), code that needs explicit index / break / continue, complex stateful traversal, when readability suffers. A `for` loop is sometimes clearer.

### Q7: `Optional` — core methods?
**A:** Create: `of`, `ofNullable`, `empty`. Unwrap: `orElse(default)`, `orElseGet(supplier)`, `orElseThrow(supplier)`, `get()` (avoid). Transform: `map`, `flatMap`, `filter`. Side-effect: `ifPresent`, `ifPresentOrElse` (Java 9+).

### Q8: `orElse` vs `orElseGet`?
**A:** `orElse(default)` evaluates `default` always — even when the Optional has a value. `orElseGet(() -> default)` evaluates only when empty. Use `orElseGet` for any non-trivial default.

### Q9: Method references — four kinds?
**A:** Static (`Integer::parseInt`), instance bound to type (`String::toUpperCase`), instance bound to a specific object (`prefix::concat`), constructor (`ArrayList::new`).

### Q10: What's `var` (Java 10)?
**A:** Local-variable type inference. Compiler infers the type from the initializer. Local variables only — not fields, parameters, or return types. Use only when the type is obvious from the right-hand side.

### Q11: Switch expressions (Java 14+)?
**A:** `switch` returns a value with arrow syntax (no fall-through). `yield` returns from a block branch. Must be exhaustive (covers all cases or has default).

### Q12: `Stream.toList()` (Java 16) vs `Collectors.toList()`?
**A:** `Stream.toList()` returns an unmodifiable List. `Collectors.toList()` returns a mutable `ArrayList`. Prefer `Stream.toList()` unless mutability is needed.

### Q13: When should you use a parallel stream?
**A:** When work is CPU-bound, the source is splittable (array, ArrayList), elements are independent, the dataset is large, and the per-element cost is high. Otherwise serial is faster.

### Q14: Why are streams lazy?
**A:** Two benefits: short-circuiting (`limit`, `findFirst`) lets you process a tiny prefix of a huge source, and the JVM can fuse multiple stages into a single pass.

### Q15: What's `Stream.peek` for?
**A:** Debugging/logging during a pipeline. Don't rely on it for side effects — short-circuiting can skip elements you'd expect to see.

### Q16: Records vs classes?
**A:** Records are auto-generated immutable data carriers (Java 16+). Final fields, ctor, accessors (`name()`), `equals`/`hashCode`/`toString`. Compact constructors for validation. Use for immutable value types. See [03 §5.3](./03_oop_language.md#53-records-java-16).

### Q17: Sealed classes — what do they enable?
**A:** Restrict the set of subclasses to a permitted list. Enables exhaustive `switch` expressions without `default`. See [03 §5.4](./03_oop_language.md#54-sealed-classes-java-17).

### Q18: Virtual threads (Java 21+)?
**A:** JVM-scheduled threads that unmount from a carrier OS thread on blocking I/O. Cheap (KB), so you can have millions. Block-style code stays simple. Don't pool them. See [01 §5.10](./01_concurrency.md#510-virtual-threads-java-21).

---

### Key Code Patterns

**Stream pipeline with grouping + downstream**
```java
Map<String, Long> byDept = employees.stream()
    .filter(e -> e.salary() > 50_000)
    .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
```

**flatMap nested lists**
```java
List<String> tags = orders.stream().flatMap(o -> o.tags().stream()).distinct().toList();
```

**Optional chain with safe default**
```java
String email = repo.findById(id).map(User::profile).map(Profile::email).orElse("unknown@x.com");
```

**Reduce to a single value**
```java
int sum = ints.stream().reduce(0, Integer::sum);
```

**Lambda as strategy**
```java
list.sort(Comparator.comparing(Employee::salary).reversed());
```

---

## 8. Self-Test

**Easy**
- [ ] What's a lambda? What's the type of a lambda?
- [ ] What's a functional interface? Name three.
- [ ] What's an intermediate vs terminal stream op?
- [ ] When use `Optional.orElse` vs `Optional.orElseGet`?

**Medium**
- [ ] Trace the §2 walkthrough — what runs when?
- [ ] Walk through `Collectors.groupingBy(dept, counting())` — what's the return type?
- [ ] Show two ways to capture a value from an enclosing scope into a lambda — and explain "effectively final".
- [ ] When is a parallel stream a bad idea?

**Hard**
- [ ] Implement the strategy pattern using a `Function` and a constructor-injected lambda.
- [ ] Why does `peek` on a short-circuited pipeline behave non-deterministically?
- [ ] Explain how `flatMap` enables the monadic chaining of `Optional`.
- [ ] What happens if you call `forEach` on the same stream twice?
- [ ] Show a switch expression that's exhaustive over a sealed hierarchy.

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Lambda** | An inline anonymous function — a value of a functional interface. |
| **Functional interface** | An interface with exactly one abstract method. |
| **Method reference** | Compact lambda that just calls another method (`String::toUpperCase`). |
| **Stream** | A pipeline of operations on a collection, evaluated lazily. |
| **Intermediate operation** | Stream stage that produces another stream (lazy). |
| **Terminal operation** | Stream stage that triggers execution and returns a non-stream result. |
| **Short-circuiting** | A stream stage that can stop processing early (`limit`, `findFirst`). |
| **Stateful operation** | A stream stage that needs to see all elements (`sorted`, `distinct`). |
| **Collector** | Object that accumulates stream elements into a result (`Collectors.toMap`). |
| **`Optional`** | Container that may or may not hold a value. |
| **Effectively final** | A local variable that is assigned once and never modified. |
| **`var`** | Java 10+ keyword for local-variable type inference. |
| **Record** | Java 16+ immutable data carrier with compiler-generated boilerplate. |
| **Sealed class** | Java 17+ class with a restricted, declared list of permitted subclasses. |
| **Pattern matching** | `instanceof`/`switch` syntax that combines type test with variable binding. |
| **Virtual thread** | Java 21+ lightweight JVM-scheduled thread on a carrier OS thread. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/06_java8_plus_doubts.md) · [← Prev: 05 Strings](./05_strings.md) · [Next: 07 JVM →](./07_jvm.md)

[↑ Back to top](#06--java-8-features)
