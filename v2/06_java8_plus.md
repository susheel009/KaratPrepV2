# 06 — Java 8+ Features

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## 🟢 Start Here — Java 8+ Features in Plain English

### What changed in Java 8?

Java 8 was a **huge update** that made Java code shorter and more readable. The three biggest additions:

### 1. Lambdas — tiny anonymous functions

Before Java 8, doing something simple (like sorting a list) required a lot of code. Lambdas let you write a function in one line.

```java
// BEFORE Java 8 — verbose
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
});

// AFTER Java 8 — one line
names.sort((a, b) -> a.length() - b.length());
// Read it as: "given a and b, return a's length minus b's length"
```

**Think of a lambda as:** a small sticky note with instructions you can pass around.

### 2. Streams — process lists like a pipeline

A stream is a way to process a collection step by step — filter, transform, collect — like an assembly line.

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Dave");

List<String> result = names.stream()          // start the pipeline
    .filter(name -> name.length() > 3)        // keep names longer than 3 chars
    .map(String::toUpperCase)                 // transform to uppercase
    .toList();                                // collect the results

// result = ["ALICE", "CHARLIE", "DAVE"]
```

**Think of it as a factory conveyor belt:** items go in one end → get filtered → get transformed → come out the other end.

### 3. Optional — no more NullPointerException

`Optional` is a container that either has a value or is empty. It forces you to think about the "no value" case instead of crashing with a `NullPointerException`.

```java
// BEFORE — dangerous
User user = findUser("123");      // might return null!
String email = user.getEmail();   // 💥 NullPointerException if user is null

// AFTER — safe
Optional<User> user = findUser("123");
String email = user
    .map(User::getEmail)           // only runs if user exists
    .orElse("unknown@email.com");  // default if user is empty
```

### What's a functional interface?

It's just an interface with **one method**. Lambdas can be used wherever a functional interface is expected.

```java
// Predicate = a function that returns true/false (used for filtering)
Predicate<String> isLong = s -> s.length() > 5;
isLong.test("Hello");   // false
isLong.test("Goodbye"); // true

// Function = takes something in, gives something out
Function<String, Integer> getLength = s -> s.length();
getLength.apply("Hello");  // 5
```

> Key takeaways: Lambdas = short inline functions. Streams = process lists step-by-step (filter→map→collect). Optional = safe way to handle "might be missing" values.

---

## 📚 Study Material

### 1. Lambda Expressions — How They Work

A lambda is shorthand for an anonymous class implementing a **functional interface** (one abstract method).

```java
// Without lambda — anonymous inner class
Comparator<String> cmp = new Comparator<String>() {
    @Override
    public int compare(String a, String b) { return a.length() - b.length(); }
};

// With lambda — same thing, less ceremony
Comparator<String> cmp = (a, b) -> a.length() - b.length();

// Lambda syntax variations:
x -> x * 2                          // single param — parens optional
(x, y) -> x + y                     // multiple params
(String x) -> x.length()            // explicit type (usually inferred)
() -> System.out.println("hello")   // no params
(x) -> { int y = x * 2; return y; } // block body — need explicit return
```

**Effectively final capture:**
```java
int factor = 3;                      // effectively final — never reassigned
// factor = 4;                       // ❌ if you uncomment this, lambda below won't compile
Function<Integer, Integer> multiply = x -> x * factor;  // captures 'factor' by value
// ⚠️ Lambdas capture VALUES of local variables, not references to them
// This avoids concurrency issues — the captured value can't change
```

### 2. Functional Interface Taxonomy

| Interface | Signature | Use | Example |
|-----------|-----------|-----|---------|
| `Function<T,R>` | `T → R` | Transform | `user -> user.getName()` |
| `Predicate<T>` | `T → boolean` | Filter/test | `user -> user.isActive()` |
| `Consumer<T>` | `T → void` | Side effect | `user -> log.info(user)` |
| `Supplier<T>` | `() → T` | Factory/lazy | `() -> new ArrayList<>()` |
| `UnaryOperator<T>` | `T → T` | Same-type transform | `s -> s.toUpperCase()` |
| `BiFunction<T,U,R>` | `(T,U) → R` | Two-arg transform | `(a,b) -> a + b` |
| `BinaryOperator<T>` | `(T,T) → T` | Two same-type → one | `Integer::sum` |

```java
// Composing functions
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> pipeline = trim.andThen(upper);  // trim first, then uppercase
pipeline.apply("  hello  ");  // "HELLO"

// Composing predicates
Predicate<Employee> senior = e -> e.getYears() > 5;
Predicate<Employee> highPay = e -> e.getSalary() > 100_000;
Predicate<Employee> seniorHighPay = senior.and(highPay);   // combined predicate
```

### 3. Streams — How the Pipeline Works

Streams are **lazy** — intermediate operations build a pipeline; nothing executes until a terminal operation is called.

```java
List<String> result = employees.stream()           // 1. SOURCE: creates Stream<Employee>
    .filter(e -> e.getSalary() > 50000)            // 2. INTERMEDIATE (lazy): no filtering yet
    .map(Employee::getName)                        // 3. INTERMEDIATE (lazy): no mapping yet
    .sorted()                                      // 4. INTERMEDIATE (stateful): needs all elements
    .limit(10)                                     // 5. INTERMEDIATE (short-circuiting)
    .collect(Collectors.toList());                 // 6. TERMINAL: NOW everything executes

// Execution model: elements flow ONE AT A TIME through the pipeline
// Employee → filter → map → sorted (buffers) → limit → collect
// NOT: filter ALL, then map ALL, then sort ALL
// Short-circuiting: limit(10) stops the pipeline after 10 elements reach it
```

**Intermediate operations (lazy):**

| Operation | Type | Description |
|-----------|------|-------------|
| `filter(pred)` | Stateless | Keep elements matching predicate |
| `map(func)` | Stateless | Transform each element |
| `flatMap(func)` | Stateless | One-to-many, then flatten |
| `distinct()` | Stateful | Remove duplicates (uses equals/hashCode) |
| `sorted()` | Stateful | Sort (must see all elements first) |
| `limit(n)` | Short-circuiting | Take first N |
| `skip(n)` | Stateful | Skip first N |
| `peek(consumer)` | Stateless | Side-effect (debugging only) |

**Terminal operations (trigger execution):**

| Operation | Returns | Short-circuiting? |
|-----------|---------|:------------------:|
| `collect(collector)` | Varies | No |
| `forEach(consumer)` | void | No |
| `reduce(identity, op)` | T | No |
| `count()` | long | No |
| `toList()` (Java 16) | Unmodifiable List | No |
| `findFirst()` | Optional | ✅ |
| `findAny()` | Optional | ✅ |
| `anyMatch(pred)` | boolean | ✅ |
| `allMatch(pred)` | boolean | ✅ |

### 4. Collectors — The Essential Ones

```java
// groupingBy — group into Map<K, List<V>>
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));

// groupingBy with downstream — aggregate within groups
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept,
             Collectors.averagingDouble(Employee::getSalary)));

// partitioningBy — split into true/false groups
Map<Boolean, List<Employee>> split = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 100_000));

// joining — concatenate strings
String names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]"));  // delimiter, prefix, suffix
// "[Alice, Bob, Charlie]"

// toMap — build a Map from stream
Map<String, Employee> byId = employees.stream()
    .collect(Collectors.toMap(Employee::getId, Function.identity(),
             (existing, replacement) -> existing));  // merge function for duplicates
```

### 5. flatMap — Flattening Nested Structures

```java
// PROBLEM: You have a list of orders, each with a list of items
List<Order> orders = getOrders();
// orders.stream().map(Order::getItems) → Stream<List<Item>>  ← nested!

// SOLUTION: flatMap flattens
List<Item> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())   // Stream<List<Item>> → Stream<Item>
    .distinct()
    .toList();

// Think of flatMap as: map + flatten
// map:     1 element → 1 element     (one-to-one)
// flatMap: 1 element → N elements    (one-to-many, flattened)
```

### 6. Optional — Correct Usage

```java
// Creating
Optional<User> user = Optional.of(obj);           // throws NPE if obj is null
Optional<User> user = Optional.ofNullable(obj);    // OK if null — becomes empty
Optional<User> empty = Optional.empty();

// Transforming (monadic operations)
String email = repo.findById(id)                   // Optional<User>
    .map(User::getProfile)                         // Optional<Profile>
    .map(Profile::getEmail)                        // Optional<String>
    .orElse("unknown@citi.com");                   // unwrap with default

// orElse vs orElseGet
opt.orElse(expensiveDefault());                    // expensiveDefault() runs ALWAYS
opt.orElseGet(() -> expensiveDefault());           // runs ONLY when empty — use this

// orElseThrow
User user = repo.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));

// ifPresent
opt.ifPresent(user -> emailService.send(user));    // only executes if present
opt.ifPresentOrElse(                               // Java 9
    user -> emailService.send(user),
    () -> log.warn("No user found")
);
```

**Anti-patterns:**
```java
// ❌ NEVER do this — defeats the purpose of Optional
if (opt.isPresent()) { return opt.get(); } else { return default; }
// ✅ DO this
return opt.orElse(default);

// ❌ Don't use Optional for fields, parameters, or collections
class User { Optional<String> name; }     // ❌
void process(Optional<String> name) {}    // ❌ — use @Nullable or overload
Optional<List<String>> names = ...;       // ❌ — return empty list instead
```

### 7. Method References — Four Types

```java
// 1. Static method: ClassName::staticMethod
Function<String, Integer> parse = Integer::parseInt;       // s -> Integer.parseInt(s)

// 2. Instance method of a type: ClassName::instanceMethod
Function<String, String> upper = String::toUpperCase;      // s -> s.toUpperCase()

// 3. Instance method of a specific object: object::method
String prefix = "Hello";
Function<String, String> greet = prefix::concat;           // s -> prefix.concat(s)

// 4. Constructor: ClassName::new
Supplier<ArrayList<String>> factory = ArrayList::new;      // () -> new ArrayList<>()
Function<String, File> fileFactory = File::new;            // s -> new File(s)
```

### 8. Modern Java Features (9-21)

```java
// Java 10: var — local variable type inference
var list = new ArrayList<String>();       // compiler infers ArrayList<String>
var stream = list.stream();              // infers Stream<String>
// Only for local variables — NOT fields, params, or return types

// Java 14: Switch expressions
String label = switch (status) {
    case ACTIVE -> "Active";              // arrow syntax — no fall-through
    case INACTIVE -> "Inactive";
    case PENDING -> {
        log.info("Pending case");
        yield "Pending";                  // yield for block body
    }
};

// Java 15: Text blocks
String sql = """
        SELECT u.name, u.email
        FROM users u
        WHERE u.active = true
        ORDER BY u.name
        """;
// Common leading whitespace stripped, trailing newline preserved

// Java 16: Stream.toList() — returns unmodifiable list
List<String> names = stream.map(User::getName).toList();  // vs Collectors.toList() (mutable)

// Java 21: Sequenced collections
SequencedCollection<String> seq = new ArrayList<>();
seq.addFirst("a"); seq.addLast("z");
seq.getFirst(); seq.getLast(); seq.reversed();
```

---

## Rapid-Fire Q&A

### Q1: What are lambdas and when do you use them?
**A:** Anonymous function syntax: `(params) -> expression`. Used wherever a functional interface is expected. Cleaner than anonymous classes. Captured variables must be effectively final.

### Q2: What's a functional interface?
**A:** Interface with exactly one abstract method. Annotated `@FunctionalInterface`. Core ones: `Function<T,R>` (T→R), `Predicate<T>` (T→boolean), `Consumer<T>` (T→void), `Supplier<T>` (→T), `BiFunction<T,U,R>`, `UnaryOperator<T>` (T→T).

### Q3: Streams — `map` vs `flatMap`?
**A:** `map`: one-to-one transformation. `flatMap`: one-to-many — flattens nested collections. E.g., `orders.stream().flatMap(o -> o.getItems().stream())` — turns `Stream<Order>` into `Stream<Item>`.

### Q4: Stream pipeline — intermediate vs terminal ops?
**A:** Intermediate (lazy): `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `peek`. Terminal (triggers execution): `collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`, `toList()`. Streams are consumed once — can't reuse.

### Q5: `Collectors.groupingBy` — how does it work?
**A:** Groups stream elements by a classifier function. Returns `Map<K, List<V>>`. Downstream collector for aggregation: `groupingBy(Employee::getDept, counting())` → `Map<String, Long>`.

### Q6: When are streams a bad choice?
**A:** When readability suffers, tight performance-critical loops (streams have overhead), when you need complex mutable state during iteration, when you need to break/continue (use loop), when the pipeline is trivially simple (loop is clearer).

### Q7: `Optional` — core methods?
**A:** `of(value)` (throws if null), `ofNullable(value)`, `empty()`. Access: `orElse(default)` (always evaluates default), `orElseGet(supplier)` (lazy), `orElseThrow()`. Transform: `map(fn)`, `flatMap(fn)`, `filter(pred)`. Never use `get()` without `isPresent()`.

### Q8: `orElse` vs `orElseGet` — subtle difference?
**A:** `orElse(expensiveCall())` — `expensiveCall()` runs **always**, even when Optional has a value. `orElseGet(() -> expensiveCall())` — only runs when empty. Use `orElseGet` for expensive defaults.

### Q9: Method references — four types?
**A:** Static: `Integer::parseInt`. Instance on type: `String::toUpperCase`. Instance on object: `myObj::getField`. Constructor: `ArrayList::new`.

### Q10: What's `var` (Java 10)?
**A:** Local variable type inference. Compiler infers the type. `var list = new ArrayList<String>()`. Only for local variables — not fields, parameters, or return types. Improves readability when the type is obvious from the right-hand side.

### Q11: Text blocks (Java 15)?
**A:** Multi-line string literals with `"""`. Strips common leading whitespace. Good for SQL, JSON, HTML templates. `String json = """ { "key": "value" } """;`

### Q12: Switch expressions (Java 14+)?
**A:** `var result = switch(day) { case MON -> "start"; case FRI -> "end"; default -> "mid"; };`. Arrow syntax (no fall-through), can return values, exhaustive with sealed classes.

### Q13: `Stream.toList()` (Java 16) vs `Collectors.toList()`?
**A:** `toList()` returns an unmodifiable list. `Collectors.toList()` returns a mutable `ArrayList`. If you need mutable, use `Collectors.toCollection(ArrayList::new)`.

---

## Key Code Patterns

```java
// Stream pipeline with grouping
Map<String, Long> countByDept = employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

// flatMap — flatten nested lists
List<String> allTags = orders.stream()
    .flatMap(order -> order.getTags().stream())
    .distinct()
    .toList();

// Optional chaining
String email = repo.findById(id)
    .map(User::getProfile)
    .map(Profile::getEmail)
    .orElse("unknown@citi.com");

// reduce
int sum = numbers.stream().reduce(0, Integer::sum);
```

---

## Can you answer these cold?

- [ ] `map` vs `flatMap` — give an example with nested collections
- [ ] Intermediate vs terminal stream operations
- [ ] `orElse` vs `orElseGet` — when the difference matters
- [ ] Four types of method references
- [ ] `Collectors.groupingBy` with downstream collector
- [ ] When streams are a bad choice — give 3 reasons

[← Back to Index](./00_INDEX.md)
