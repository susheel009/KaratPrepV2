# 06 — Java 8+ Features

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

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
