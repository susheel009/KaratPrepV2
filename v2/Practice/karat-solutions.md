# Karat Java Practice — Solutions

Solutions to the 20 debugging problems and 10 ladder problems. **Don't read this until you've attempted the problem.**

For debugging problems, each solution gives:
- **Output** (what the code prints, or what exception is thrown)
- **Mechanism** (the precise terminology to use in the interview)
- **Exact exception class** where applicable
- **Fix pattern**

For ladder problems, each solution gives:
- **Bug location** (which line/method)
- **Root cause** (what was logically wrong)
- **Fix** (the corrected method)
- **Task 2 implementation** (the requested extension)
- **Task 3 implementation** (where applicable)

## Index

### Debugging Solutions
| # | Title | Quick answer |
|---|---|---|
| [DBG-01](#dbg-01-solution) | Static List Accumulator | List grows across calls — shared static state |
| [DBG-02](#dbg-02-solution) | Integer Cache Boundary | true / false / true |
| [DBG-03](#dbg-03-solution) | String Pool Reference | true / false / true / true |
| [DBG-04](#dbg-04-solution) | Map Lookup Auto-Unbox | NullPointerException |
| [DBG-05](#dbg-05-solution) | String + Int Concatenation | "Result: 53" / "Result: 8" / "8 items" |
| [DBG-06](#dbg-06-solution) | printf Format Mismatch | IllegalFormatConversionException |
| [DBG-07](#dbg-07-solution) | Remove While Iterating | ConcurrentModificationException |
| [DBG-08](#dbg-08-solution) | Lambda in Loop Capture | 0 1 2 (with snapshot); compile error without |
| [DBG-09](#dbg-09-solution) | Try-With-Resources Order | Reverse close order |
| [DBG-10](#dbg-10-solution) | Finally Overrides Return | 2 / 99 |
| [DBG-11](#dbg-11-solution) | Switch Without Break | "two three other" |
| [DBG-12](#dbg-12-solution) | Integer Division | 2.0 2.5 2.5 |
| [DBG-13](#dbg-13-solution) | Floating-Point Equality | false / true |
| [DBG-14](#dbg-14-solution) | Shallow Copy of Nested List | Inner lists shared, outer not |
| [DBG-15](#dbg-15-solution) | Arrays.asList Add | UnsupportedOperationException |
| [DBG-16](#dbg-16-solution) | Static Method "Override" | "generic" / "bark" — method hiding |
| [DBG-17](#dbg-17-solution) | String split Trailing | trailing empties trimmed by default |
| [DBG-18](#dbg-18-solution) | Stream Reduce From Loop | 56 56 |
| [DBG-19](#dbg-19-solution) | final Keyword | reassign const, subclass final, override final all fail |
| [DBG-20](#dbg-20-solution) | Volatile Counter Race | not reliable — `count++` is not atomic |
| [DBG-21](#dbg-21-solution) | parseInt Whitespace | only "42" succeeds |
| [DBG-22](#dbg-22-solution) | PECS Wildcard | extends ⇒ read; super ⇒ write |
| [DBG-23](#dbg-23-solution) | Mutable HashMap Key | hashCode/equals contract violation |

### Ladder Solutions
| # | Bug | Extension |
|---|---|---|
| [LADDER-01](#ladder-01-solution) | Missing `!l.returned` filter | Frequency map for mostBorrowedBook |
| [LADDER-02](#ladder-02-solution) | Filtered PENDING instead of PAID | groupingBy category + min comparator |
| [LADDER-03](#ladder-03-solution) | Summed durationSeconds instead of repetitions | groupingBy + averagingInt |
| [LADDER-04](#ladder-04-solution) | Counted ALL reservations as occupying seats | groupingBy route + sum pricePaid |
| [LADDER-05](#ladder-05-solution) | Counted only CORRECT (gym-membership pattern) | per-question incorrect rate |
| [LADDER-06](#ladder-06-solution) | Didn't filter to complete journeys | groupingBy driver + summingInt |
| [LADDER-07](#ladder-07-solution) | Used basePrice instead of effectiveUnitPrice | nested loop with min per product |
| [LADDER-08](#ladder-08-solution) | Returned events.size() unconditionally | filter completed + groupingBy genre |
| [LADDER-09](#ladder-09-solution) | Didn't filter for POSTED status | filter POSTED + DEBIT, groupingBy category |
| [LADDER-10](#ladder-10-solution) | Didn't filter cancelled bookings | groupingBy room + Monte Carlo simulation |

---

# Debugging Solutions

## DBG-01 Solution

**Output:**
```
[10]
[10, 11]
[10, 11, 12]
```

**Mechanism**: `items` is a `static` field — shared across all calls to `add`. Each call mutates the same list. This is the Java equivalent of Python's mutable-default-argument gotcha. The same instance is mutated and returned every time.

**Fix**: avoid shared mutable static state. If a method needs a per-call list, allocate one inside the method:
```java
public static List<Integer> add(Integer x) {
    List<Integer> result = new ArrayList<>();
    result.add(x);
    return result;
}
```
Or, if the registry behavior is intentional, return a defensive copy: `return new ArrayList<>(items);`

---

## DBG-02 Solution

**Output:**
```
true
false
true
```

**Mechanism**: Java caches `Integer` objects for values `-128` to `127`. When you autobox `127`, both `a` and `b` reference the same cached `Integer`, so `==` (reference comparison) returns `true`. `128` is outside the cache; each autoboxing creates a new `Integer`, so `c == d` is `false`. `equals` always compares values, so it's `true`.

**Rule**: never use `==` to compare `Integer` objects. Always `.equals()` (or unbox with `intValue()`).

---

## DBG-03 Solution

**Output:**
```
true
false
true
true
```

**Mechanism**: String literals (`"hello"`) are interned in the **string pool**. Two literals with the same content share a reference, so `x == y` is `true`. `new String("hello")` allocates a fresh object on the heap, so `x == z` is `false`. `equals` compares values: `true`. `intern()` returns the canonical pooled reference, so `x == z.intern()` is `true`.

This is the Java equivalent of Python's `is` vs `==`.

**Rule**: always use `.equals()` to compare strings.

---

## DBG-04 Solution

**Output**: throws `NullPointerException` on the `int bananaCount = counts.get("banana");` line.

**Mechanism**: `Map.get` returns `null` when the key is missing. The compiler tries to **auto-unbox** `null` into `int`, which calls `null.intValue()` — that throws NPE.

**Fix idiomatically**:
```java
int bananaCount = counts.getOrDefault("banana", 0);
```

---

## DBG-05 Solution

**Output:**
```
Result: 53
Result: 8
8 items
```

**Mechanism**: `+` is **left-associative**. In line 1, `"Result: " + 5` evaluates first to the string `"Result: 5"`, then `+ 3` concatenates `"3"` giving `"Result: 53"`. Parentheses in line 2 force `5 + 3` (integer addition) to evaluate first, then concat. In line 3, the leftmost `5 + 3` is integer addition (both operands are int), then `+ " items"` triggers concat.

**Rule**: when mixing strings and numbers in `+`, group with parentheses to make intent explicit.

---

## DBG-06 Solution

**Output**: line 1 prints `Name: Alice, Age: 30`. Line 2 throws `IllegalFormatConversionException` (specifically: `IllegalFormatConversionException: d != java.lang.String`).

**Mechanism**: `%d` requires an `Integer` (or compatible primitive). Passing a `String` violates the format specifier contract.

**Fix**: pass the right types, or use `%s` for everything if you don't care about formatting:
```java
System.out.printf("Name: %s, Age: %s%n", name, "30");
```

---

## DBG-07 Solution

**Output**: throws `java.util.ConcurrentModificationException` from the iterator's `next()` call.

**Mechanism**: Enhanced for-loop uses an `Iterator` under the hood. `ArrayList`'s iterator tracks a `modCount`; calling `nums.remove(n)` increments `modCount` without going through the iterator. On the next `iterator.next()` call, the iterator detects the mismatch and throws.

**Fixes** (any of):
1. **`Iterator.remove()`** — manual iterator, use its own remove method:
```java
Iterator<Integer> it = nums.iterator();
while (it.hasNext()) {
    if (it.next() % 2 == 0) it.remove();
}
```
2. **`removeIf`** (Java 8+):
```java
nums.removeIf(n -> n % 2 == 0);
```
3. **Iterate a copy**:
```java
for (Integer n : new ArrayList<>(nums)) {
    if (n % 2 == 0) nums.remove(n);
}
```

---

## DBG-08 Solution

**Output (with `snapshot`):** `0 1 2 `.

**Without `snapshot`** (i.e., capturing `i` directly): the code wouldn't compile. Compile error: *"local variables referenced from a lambda expression must be final or effectively final"*.

**Mechanism**: Java requires lambda-captured local variables to be effectively final. The loop variable `i` is reassigned each iteration, so it's not effectively final. By assigning `int snapshot = i` inside the loop body, you create a new local each iteration that IS effectively final. This is Java's safer answer to Python's late-binding-closure gotcha — Java catches the bug at compile time.

---

## DBG-09 Solution

**Output:**
```
Open A
Open B
Use both
Close B
Close A
```

**Mechanism**: try-with-resources closes resources **in reverse declaration order** — last opened, first closed. This matches stack-based scoping.

**If `b.close()` throws while the try block is also throwing**: the original exception propagates and the close exception becomes a **suppressed exception** attached to it (accessible via `Throwable.getSuppressed()`). The original exception "wins."

---

## DBG-10 Solution

**Output:**
```
2
99
```

**Mechanism**: a `return` (or `throw`) in a `finally` block **overrides** any pending return or exception from the `try`. In `compute()`, the `try` returns 1 but the `finally` returns 2, so 2 wins. In `compute2()`, the try throws but the finally's `return 99` swallows the exception and the method returns 99.

**Why anti-pattern**: it silently discards exceptions and pending returns, making debugging miserable. **Never `return` from a `finally` block.**

---

## DBG-11 Solution

**Output:**
```
two
three
other
```

**Mechanism**: classic **switch fall-through**. Without `break`, after the matching case (`case 2`) executes, control falls through to subsequent cases, including `default`.

**Java 14+ arrow form** prevents this by design:
```java
switch (x) {
    case 1 -> System.out.println("one");
    case 2 -> System.out.println("two");
    case 3 -> System.out.println("three");
    default -> System.out.println("other");
}
```
Each arrow case is independent — no fall-through possible.

---

## DBG-12 Solution

**Output:** `2.0 2.5 2.5`

**Mechanism**: in `r1`, `a / b` is **integer division** because both operands are `int` — result is `2`, then promoted to `double` giving `2.0`. The cast in `r2` converts `a` to `double` first, so the division is `double / int`, which promotes to `double / double` and gives `2.5`. Same in `r3` with `b` cast.

**Rule**: cast at least one operand to `double` *before* division, or arithmetic happens in integer space.

---

## DBG-13 Solution

**Output:**
```
0.30000000000000004
false
true
```

**Mechanism**: doubles use IEEE 754 binary floating-point. Decimals like `0.1` and `0.2` can't be represented exactly in binary, so their sum has a tiny error. `==` is exact, so it returns `false`.

**Correct comparison**: `Math.abs(a - b) < epsilon` for some small epsilon. For financial calculations, use `BigDecimal` (constructor with `String`, not `double`).

---

## DBG-14 Solution

**Output:**
```
original: [[1, 2, 99], [3, 4]]
shallow:  [[1, 2, 99], [3, 4], [5, 6]]
```

**Mechanism**: `new ArrayList<>(original)` is a **shallow copy** — it creates a new outer list, but each element of the outer list is a reference to the same inner list as in `original`. So:
- `shallow.get(0).add(99)` mutates the inner list — visible in both.
- `shallow.add(...)` adds to the new outer list — not visible in `original`.

**Deep copy**: copy each inner list too:
```java
List<List<Integer>> deep = original.stream()
    .map(ArrayList::new)
    .collect(Collectors.toList());
```

---

## DBG-15 Solution

**Output:** `[99, 2, 3]`, then throws `UnsupportedOperationException` on `list.add(4)`.

**Mechanism**: `Arrays.asList(1, 2, 3)` returns a **fixed-size view** backed by the original varargs array. `set` works because it modifies an existing slot. `add` would require resizing — not supported.

**Fix**: wrap in a real `ArrayList`:
```java
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3));
```

Note: `List.of(1, 2, 3)` (Java 9+) is fully **immutable** — neither `set` nor `add` works.

---

## DBG-16 Solution

**Output:**
```
generic
bark
```

**Mechanism**: static methods are **hidden**, not **overridden**. Static method calls are resolved at **compile time** based on the **declared (static) type** of the variable, not the runtime type. `a` is declared as `Animal`, so `a.sound()` calls `Animal.sound()`. `d` is declared as `Dog`, so `d.sound()` calls `Dog.sound()`.

**Rule**: never call static methods on instance variables — it misleadingly looks polymorphic. Always call them on the class: `Animal.sound()`.

---

## DBG-17 Solution

**Output:**
```
[a, b, c]
[a, , b]
[a]
[a, , ]
```

**Mechanism**: `String.split(regex)` is shorthand for `split(regex, 0)`. With limit `0` (default), **trailing empty strings are removed** from the result. With limit `-1`, no removal — all matches preserved. Embedded empties (between two delimiters) are always preserved.

**Rule**: when round-tripping data through split/join, use `split(regex, -1)` to preserve trailing empties.

---

## DBG-18 Solution

**Output:** `56 56`

**Mechanism**: the stream pipeline filters even numbers (2, 4, 6), maps each to its square (4, 16, 36), and sums them: 4+16+36 = 56. The for-loop is the equivalent imperative version.

**Stream pipeline operations by category:**
- `filter(Predicate)` — intermediate, lazy
- `mapToInt(ToIntFunction)` — intermediate, returns IntStream
- `sum()` — terminal, eager (triggers evaluation)

---

## DBG-19 Solution

**Three commented lines and why each fails:**

1. `CONSTANT = 100;` — fails to compile. `final` variables can't be reassigned: *"cannot assign a value to final variable CONSTANT."*
2. `class IceCold extends Frozen {}` — fails to compile. `final` class can't be subclassed: *"cannot inherit from final Frozen."*
3. `(override doIt() in subclass)` — fails to compile. `final` method can't be overridden: *"doIt() in Subclass cannot override doIt() in FinalThreeWays — overridden method is final."*

**Why `f.name = "changed"` works**: `final` on the **class** prevents inheritance, not internal state mutation. The field `name` itself is not declared `final`, so it's mutable. To freeze fields, mark them `final` individually or make the class properly immutable.

---

## DBG-20 Solution

**Output**: probably `20000`, but **not reliably**. May print 19945, 19987, etc. (Run it 5 times and you'll likely see at least one wrong value.)

**Mechanism**: `volatile` guarantees **visibility** (each thread sees the latest value of `count`) and **ordering** (`volatile` writes have happens-before semantics with subsequent reads). But `count++` is **read-modify-write**: load count, add 1, store. Three steps. Two threads can both load the same value, both add 1, both store — losing one increment. `volatile` does NOT make compound operations atomic.

**Fixes** (any of):
1. **`AtomicInteger`** with `incrementAndGet()` — lock-free, uses CAS.
2. **`synchronized`** block around the increment.
3. **`LongAdder`** for high-contention scenarios — better than AtomicInteger when many threads contend.

---

## DBG-21 Solution

**Output**: `42`, then three `NumberFormatException`s for `" 42"`, `"42L"`, and `""`.

**Mechanism**: `Integer.parseInt` is strict — no whitespace tolerance, no suffix tolerance, no empty string. Any deviation throws `NumberFormatException`.

**Safer pattern**:
```java
int parsed = Integer.parseInt(input.trim());
```
Or for fully defensive parsing:
```java
try {
    return Integer.parseInt(input.trim());
} catch (NumberFormatException e) {
    return defaultValue;
}
```

---

## DBG-22 Solution

| Line | Compiles? | Why |
|---|---|---|
| `nums.add(4)` | No | `? extends Number` is a **producer** — you can read but not write. Compiler can't verify `4` (Integer) is the same as the unknown `?` type. |
| `nums.add((Integer) 4)` | No | Same reason. The cast doesn't help; the wildcard restricts writes regardless. |
| `Number first = nums.get(0)` | Yes | Reading from `? extends Number` gives you `Number` (or subtype), assignable to `Number`. |
| `sink.add(5)` | Yes | `? super Integer` is a **consumer** — you can write `Integer` (or subtype) to it. The list is *some* supertype of Integer, so adding Integer is safe. |
| `Integer x = sink.get(0)` | No | Reading from `? super Integer` gives you `Object` (the most you know — could be Number, Object, etc.). Not assignable to `Integer` without cast. |

**Rule**: PECS — **Producer Extends, Consumer Super.** If you only **read** from it, use `? extends T`. If you only **write** to it, use `? super T`.

---

## DBG-23 Solution

**Output:**
```
first
null   (or sometimes "first" — implementation-dependent)
1
```

**Mechanism**: `HashMap` stores entries in buckets indexed by `key.hashCode() % capacity`. When you `put(key, "first")` with key `[1,2,3]`, it's placed in bucket A. When you mutate the key by adding `4`, the list's `hashCode()` changes — so `map.get(key)` now hashes to bucket B. Bucket B is empty, so it returns `null`. The entry in bucket A is orphaned: it's still there (size is 1), but **unreachable** because no key hashes back to it.

**Rule**: keys in a `HashMap` should be **effectively immutable** — or at least, their `hashCode()` and `equals()` results must not change after insertion. Common mistake: using `List`, `Set`, or any mutable collection as a key. Use immutable types (`String`, primitive wrappers, `record`, defensive copies).

---

# Ladder Solutions

## LADDER-01 Solution

### Bug

`totalActiveLoans()` counts all loans, ignoring the `returned` flag.

```java
public int totalActiveLoans() {
    return (int) loans.stream().count();  // BUG: counts everything
}
```

### Fix

```java
public int totalActiveLoans() {
    return (int) loans.stream().filter(l -> !l.returned).count();
}
```

The signal in the test data: 5 loans total, 3 are active (l2, l4, l5 — the ones without `markReturned()`). The buggy version returns 5, the test expects 3.

### Task 2: `mostBorrowedBook()`

```java
public Book mostBorrowedBook() {
    if (loans.isEmpty()) return null;
    Map<Book, Long> counts = loans.stream()
        .collect(Collectors.groupingBy(l -> l.book, Collectors.counting()));
    return counts.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElse(null);
}
```

Test:
```java
assert reg.mostBorrowedBook().equals(b1) :
    "mostBorrowedBook should be b1, was " + reg.mostBorrowedBook().title;
```

In the test data, b1 appears in 3 loans (l1, l3, l5), b2 once, b3 once. So b1 wins.

### Task 3: `averageLoanDuration()`

```java
public double averageLoanDuration() {
    return loans.stream()
        .filter(l -> l.returned)
        .mapToInt(Loan::getDuration)
        .average()
        .orElse(0.0);
}
```

---

## LADDER-02 Solution

### Bug

`totalRevenue()` filters `PENDING` instead of `PAID`.

```java
.filter(o -> o.status == OrderStatus.PENDING)  // BUG: wrong status
```

### Fix

```java
.filter(o -> o.status == OrderStatus.PAID)
```

In the test, paid orders are o1 (12.0) and o3 (15.0), totaling 27.0.

### Task 2: `cheapestItemPerCategory()`

```java
public Map<String, MenuItem> cheapestItemPerCategory() {
    return orders.stream()
        .filter(o -> o.status == OrderStatus.PAID)
        .flatMap(o -> o.items.stream())
        .collect(Collectors.toMap(
            item -> item.category,
            item -> item,
            (a, b) -> a.price <= b.price ? a : b
        ));
}
```

The merge function `(a, b) -> a.price <= b.price ? a : b` keeps the cheaper item when two items share a category.

### Task 3: `bestRevenueDay()`

```java
public int bestRevenueDay() {
    Map<Integer, Double> revByDay = orders.stream()
        .filter(o -> o.status == OrderStatus.PAID)
        .collect(Collectors.groupingBy(
            o -> o.dayPlaced,
            Collectors.summingDouble(Order::total)
        ));
    return revByDay.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElse(-1);
}
```

---

## LADDER-03 Solution

### Bug

`totalRepetitionsAllExercises()` sums `e.durationSeconds` instead of `e.repetitions`.

```java
total += e.durationSeconds;  // BUG: wrong field
```

### Fix

```java
total += e.repetitions;
```

Test expects: completed workouts are w1 (10+1=11) and w3 (20+15=35). Total = 46.

### Task 2: `averageDurationByExercise()`

```java
public Map<String, Double> averageDurationByExercise() {
    return workouts.stream()
        .filter(w -> w.completed)
        .flatMap(w -> w.exercises.stream())
        .collect(Collectors.groupingBy(
            e -> e.name,
            Collectors.averagingInt(e -> e.durationSeconds)
        ));
}
```

For the test data, "squat" completed appearances: 60s in w1, 65s in w3 → avg 62.5. "plank": 90 → avg 90. "pushup": 45 → avg 45.

### Task 3: `longestStreakDays()`

```java
public int longestStreakDays() {
    List<Integer> days = workouts.stream()
        .filter(w -> w.completed)
        .map(w -> w.day)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
    if (days.isEmpty()) return 0;
    int longest = 1, current = 1;
    for (int i = 1; i < days.size(); i++) {
        if (days.get(i) == days.get(i - 1) + 1) current++;
        else current = 1;
        longest = Math.max(longest, current);
    }
    return longest;
}
```

---

## LADDER-04 Solution

### Bug

`availableSeats()` subtracts ALL reservations from totalSeats — including waitlisted and cancelled ones, which don't occupy seats.

```java
return totalSeats - reservations.size();  // BUG
```

### Fix

```java
public int availableSeats() {
    long confirmed = reservations.stream()
        .filter(r -> r.status == ReservationStatus.CONFIRMED)
        .count();
    return totalSeats - (int) confirmed;
}
```

Signal in test data: 4 reservations but only 2 are confirmed (r1, r2). Buggy version returns 10-4=6; test expects 10-2=8.

### Task 2: `revenueByRoute()`

```java
public Map<String, Double> revenueByRoute() {
    return flights.stream()
        .flatMap(f -> f.reservations.stream()
            .filter(r -> r.status == ReservationStatus.CONFIRMED)
            .map(r -> Map.entry(f.getRoute(), r.pricePaid)))
        .collect(Collectors.groupingBy(
            Map.Entry::getKey,
            Collectors.summingDouble(Map.Entry::getValue)
        ));
}
```

### Task 3: `bestSellingClass()`

```java
public TravelClass bestSellingClass() {
    Map<TravelClass, Long> counts = flights.stream()
        .flatMap(f -> f.reservations.stream())
        .filter(r -> r.status == ReservationStatus.CONFIRMED)
        .collect(Collectors.groupingBy(r -> r.travelClass, Collectors.counting()));
    return counts.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElse(null);
}
```

---

## LADDER-05 Solution

This is the **closest direct analog** to the Karat gym membership question — counting only one valid status when two should count.

### Bug

```java
if (s == AnswerStatus.CORRECT) {
    attempted++;
}
```

Only counts CORRECT. INCORRECT also means "attempted." Identical structure to the gym bug that counted only GOLD as paid (forgetting SILVER).

### Fix

```java
if (s == AnswerStatus.CORRECT || s == AnswerStatus.INCORRECT) {
    attempted++;
}
```

In the test: 6 total answers. CORRECT + INCORRECT = 4 (a1.q1, a2.q1, a2.q2, a3.q1). 4/6 = 0.6667.

### Task 2: `hardestQuestion()`

```java
public Question hardestQuestion() {
    Map<Question, int[]> counts = new HashMap<>();  // [correct, incorrect]
    for (Attempt a : attempts) {
        for (Map.Entry<Question, AnswerStatus> e : a.answers.entrySet()) {
            counts.computeIfAbsent(e.getKey(), k -> new int[2]);
            if (e.getValue() == AnswerStatus.CORRECT) counts.get(e.getKey())[0]++;
            else if (e.getValue() == AnswerStatus.INCORRECT) counts.get(e.getKey())[1]++;
        }
    }
    Question hardest = null;
    double worstRate = -1;
    for (Map.Entry<Question, int[]> e : counts.entrySet()) {
        int correct = e.getValue()[0], incorrect = e.getValue()[1];
        if (correct + incorrect == 0) continue;
        double rate = (double) incorrect / (correct + incorrect);
        if (rate > worstRate) {
            worstRate = rate;
            hardest = e.getKey();
        }
    }
    return hardest;
}
```

### Task 3: `mostImprovedStudent()`

```java
public int mostImprovedStudent() {
    Map<Integer, List<Attempt>> byStudent = attempts.stream()
        .collect(Collectors.groupingBy(a -> a.studentId));
    
    int bestStudent = -1;
    double bestImprovement = Double.NEGATIVE_INFINITY;
    for (Map.Entry<Integer, List<Attempt>> e : byStudent.entrySet()) {
        List<Attempt> sorted = e.getValue().stream()
            .sorted(Comparator.comparingInt(a -> a.dayTaken))
            .collect(Collectors.toList());
        if (sorted.size() < 2) continue;
        Attempt first = sorted.get(0);
        Attempt last = sorted.get(sorted.size() - 1);
        double accuracyDelta = accuracy(last) - accuracy(first);
        if (accuracyDelta > bestImprovement) {
            bestImprovement = accuracyDelta;
            bestStudent = e.getKey();
        }
    }
    return bestStudent;
}

private double accuracy(Attempt a) {
    long c = a.answers.values().stream().filter(s -> s == AnswerStatus.CORRECT).count();
    long i = a.answers.values().stream().filter(s -> s == AnswerStatus.INCORRECT).count();
    return c + i == 0 ? 0.0 : (double) c / (c + i);
}
```

---

## LADDER-06 Solution

### Bug

`totalDistanceLogged()` doesn't filter to completed journeys.

```java
return journeys.stream()
    .mapToInt(Journey::totalDistance)
    .sum();   // BUG: no .filter(j -> j.complete)
```

### Fix

```java
public int totalDistanceLogged() {
    return journeys.stream()
        .filter(j -> j.complete)
        .mapToInt(Journey::totalDistance)
        .sum();
}
```

This is structurally identical to the Karat obstacle course `personalBest()` bug — same "missing complete filter" pattern. Test: j1 (250) + j3 (250) = 500; j2 is incomplete and excluded.

### Task 2: `totalDistancePerDriver()`

```java
public Map<String, Integer> totalDistancePerDriver() {
    return journeys.stream()
        .filter(j -> j.complete)
        .collect(Collectors.groupingBy(
            j -> j.driver,
            Collectors.summingInt(Journey::totalDistance)
        ));
}
```

### Task 3: `averageSpeedPerDriver()`

```java
public Map<String, Double> averageSpeedPerDriver() {
    return journeys.stream()
        .filter(j -> j.complete && j.totalTime() > 0)
        .collect(Collectors.groupingBy(
            j -> j.driver,
            Collectors.averagingDouble(j -> (double) j.totalDistance() / j.totalTime())
        ));
}
```

---

## LADDER-07 Solution

### Bug

`subtotal()` uses `basePrice` instead of `effectiveUnitPrice()` (which applies discounts).

```java
sum += line.product.basePrice * line.quantity;  // BUG: ignores discount
```

### Fix

```java
public double subtotal() {
    return lines.stream().mapToDouble(CartLine::lineTotal).sum();
}
```

Test: 2 books at 50% off = 2 × 10 = 20; 5 pens at 0% off = 10. Total 30.

### Task 2: `bestPricePerProduct()` (the bestOfBests analog)

```java
public Map<String, Double> bestPricePerProduct() {
    Map<String, Double> best = new HashMap<>();
    for (Cart cart : carts) {
        if (cart.status != CartStatus.CHECKED_OUT) continue;
        for (CartLine line : cart.lines) {
            String name = line.product.name;
            double price = line.effectiveUnitPrice();
            best.merge(name, price, Math::min);
        }
    }
    return best;
}
```

The `merge` call with `Math::min` is the elegant Java-8 way to maintain a running minimum per key — analogous to the per-obstacle minimum in `bestOfBests`.

### Task 3: `mostPopularBundle()` (pair analysis)

```java
public List<String> mostPopularBundle() {
    Map<Set<String>, Integer> pairCount = new HashMap<>();
    for (Cart cart : carts) {
        if (cart.status != CartStatus.CHECKED_OUT) continue;
        List<String> names = cart.lines.stream()
            .map(l -> l.product.name)
            .distinct()
            .collect(Collectors.toList());
        for (int i = 0; i < names.size(); i++) {
            for (int j = i + 1; j < names.size(); j++) {
                Set<String> pair = new HashSet<>(Arrays.asList(names.get(i), names.get(j)));
                pairCount.merge(pair, 1, Integer::sum);
            }
        }
    }
    return pairCount.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(e -> new ArrayList<>(e.getKey()))
        .orElse(Collections.emptyList());
}
```

---

## LADDER-08 Solution

### Bug

`totalCompletedPlays()` returns total events without checking `isCompleted()`.

```java
return events.size();  // BUG
```

### Fix

```java
public int totalCompletedPlays() {
    return (int) events.stream().filter(PlayEvent::isCompleted).count();
}
```

Test: 5 events, 3 completed (s1@230, s2@360, s3@195).

### Task 2: `topGenreByCompletedPlays()`

```java
public String topGenreByCompletedPlays() {
    if (events.isEmpty()) return null;
    Map<String, Long> counts = events.stream()
        .filter(PlayEvent::isCompleted)
        .collect(Collectors.groupingBy(e -> e.song.genre, Collectors.counting()));
    return counts.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElse(null);
}
```

### Task 3: `mostSkippedSong()`

```java
public Song mostSkippedSong() {
    Map<Song, long[]> counts = new HashMap<>();  // [completed, total]
    for (PlayEvent e : events) {
        counts.computeIfAbsent(e.song, k -> new long[2]);
        counts.get(e.song)[1]++;
        if (e.isCompleted()) counts.get(e.song)[0]++;
    }
    Song result = null;
    double worstRate = -1;
    for (Map.Entry<Song, long[]> e : counts.entrySet()) {
        long completed = e.getValue()[0], total = e.getValue()[1];
        if (total < 3) continue;
        double skipRate = (double) (total - completed) / total;
        if (skipRate > worstRate) {
            worstRate = skipRate;
            result = e.getKey();
        }
    }
    return result;
}
```

---

## LADDER-09 Solution

### Bug

`balance()` doesn't filter for `POSTED` status — it sums all transactions including PENDING and REVERSED.

```java
for (Transaction t : transactions) {
    // BUG: no status check
    if (t.type == TransactionType.CREDIT) bal += t.amount;
    else bal -= t.amount;
}
```

### Fix

```java
public double balance() {
    double bal = 0;
    for (Transaction t : transactions) {
        if (t.status != TransactionStatus.POSTED) continue;
        if (t.type == TransactionType.CREDIT) bal += t.amount;
        else bal -= t.amount;
    }
    return bal;
}
```

Test data: t1 (+1000 posted), t2 (−200 posted), t3 (−50 pending — skip), t4 (−100 reversed — skip). Balance = 1000−200 = 800.

### Task 2: `highestSpendingCategory()`

```java
public String highestSpendingCategory() {
    Map<String, Double> totals = accounts.stream()
        .flatMap(a -> a.transactions.stream())
        .filter(t -> t.status == TransactionStatus.POSTED)
        .filter(t -> t.type == TransactionType.DEBIT)
        .collect(Collectors.groupingBy(
            t -> t.category,
            Collectors.summingDouble(t -> t.amount)
        ));
    return totals.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElse(null);
}
```

### Task 3: `monthOverMonthChange()`

```java
public Map<String, Double> monthOverMonthChange() {
    List<Integer> months = accounts.stream()
        .flatMap(a -> a.transactions.stream())
        .filter(t -> t.status == TransactionStatus.POSTED)
        .map(t -> t.month)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
    if (months.size() < 2) return Collections.emptyMap();
    int recent = months.get(months.size() - 1);
    int prior  = months.get(months.size() - 2);
    
    Map<String, Double> recentTotals = sumByCategory(recent);
    Map<String, Double> priorTotals = sumByCategory(prior);
    
    Set<String> categories = new HashSet<>();
    categories.addAll(recentTotals.keySet());
    categories.addAll(priorTotals.keySet());
    
    Map<String, Double> result = new HashMap<>();
    for (String cat : categories) {
        result.put(cat, recentTotals.getOrDefault(cat, 0.0)
                       - priorTotals.getOrDefault(cat, 0.0));
    }
    return result;
}

private Map<String, Double> sumByCategory(int month) {
    return accounts.stream()
        .flatMap(a -> a.transactions.stream())
        .filter(t -> t.status == TransactionStatus.POSTED && t.type == TransactionType.DEBIT && t.month == month)
        .collect(Collectors.groupingBy(t -> t.category, Collectors.summingDouble(t -> t.amount)));
}
```

---

## LADDER-10 Solution

### Bug

`confirmedRoomNights()` doesn't filter for confirmed status.

```java
return bookings.stream()
    .mapToInt(Booking::nights)
    .sum();  // BUG: includes cancelled
```

### Fix

```java
public int confirmedRoomNights() {
    return bookings.stream()
        .filter(b -> b.status == BookingStatus.CONFIRMED)
        .mapToInt(Booking::nights)
        .sum();
}
```

Test: b1 (3) + b3 (4) + b4 (2) = 9. b2 is cancelled.

### Task 2: `mostBookedRoom()`

```java
public Room mostBookedRoom() {
    if (bookings.isEmpty()) return null;
    Map<Room, Long> counts = bookings.stream()
        .filter(b -> b.status == BookingStatus.CONFIRMED)
        .collect(Collectors.groupingBy(b -> b.room, Collectors.counting()));
    return counts.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElse(null);
}
```

### Task 3: `chanceOfFullOccupancyOnDay(int day, int trials)` — Monte Carlo

This is the analog of `chanceOfPersonalBest`. The articulation matters here — even if you don't finish the code, talking through the simulation pattern out loud scores points.

```java
public double chanceOfFullOccupancyOnDay(int day, int trials) {
    // For each room, build the pool of historical occupancy durations
    // (only from confirmed bookings of that room).
    Map<Room, List<Integer>> nightsPool = new HashMap<>();
    for (Room r : rooms) {
        List<Integer> pool = bookings.stream()
            .filter(b -> b.status == BookingStatus.CONFIRMED && b.room.equals(r))
            .map(Booking::nights)
            .collect(Collectors.toList());
        nightsPool.put(r, pool);
    }
    
    // For each room, build the pool of historical check-in days.
    Map<Room, List<Integer>> startPool = new HashMap<>();
    for (Room r : rooms) {
        List<Integer> pool = bookings.stream()
            .filter(b -> b.status == BookingStatus.CONFIRMED && b.room.equals(r))
            .map(b -> b.checkInDay)
            .collect(Collectors.toList());
        startPool.put(r, pool);
    }
    
    Random rng = new Random();
    int successes = 0;
    
    trial: for (int t = 0; t < trials; t++) {
        for (Room r : rooms) {
            List<Integer> nights = nightsPool.get(r);
            List<Integer> starts = startPool.get(r);
            if (nights.isEmpty() || starts.isEmpty()) continue trial;
            
            int simStart = starts.get(rng.nextInt(starts.size()));
            int simNights = nights.get(rng.nextInt(nights.size()));
            int simEnd = simStart + simNights;
            
            // If this room is NOT occupied on `day`, this trial fails.
            if (!(day >= simStart && day < simEnd)) continue trial;
        }
        successes++;
    }
    
    return (double) successes / trials;
}
```

**The talking-out-loud articulation** (this matters more than the code):

> "I want to estimate the probability that all rooms are occupied on the target day. I'll simulate by, for each room, randomly drawing a start day from past confirmed start days for that room and a duration from past confirmed durations. Each trial: draw simulated check-in/check-out for every room; check if every room covers the target day. Run 10,000 trials; the success fraction is the answer.
>
> Edge case: if a room has zero historical confirmed bookings, that room can't be occupied on any day, so the trial fails — using `continue trial` to skip ahead.
>
> The two pools — durations and start days — are pre-computed once outside the trial loop for efficiency."

---

# Final Notes

For each problem, after reviewing the solution:

1. **Re-attempt within 24 hours.** Solutions you peeked at don't stick unless you re-solve from scratch.
2. **Note the technical term**, especially for debugging problems. "Method hiding," "string pool," "happens-before," "PECS" — those are the words the interviewer is grading.
3. **For ladder problems, identify the bug pattern**: missing filter, wrong filter, wrong field. The same patterns repeat. Recognizing the pattern in the first read of the failing test is half the speed gain.

---

**← [Back to Master Index](karat-practice-INDEX.md)** | **[Debugging Problems](karat-debugging-problems.md)** | **[Ladder Problems](karat-ladder-problems.md)**
