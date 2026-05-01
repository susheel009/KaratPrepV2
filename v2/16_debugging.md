# 16 — Bug Taxonomy & Debugging

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## Bug Type Taxonomy

When you spot a bug in Karat, **name the type**. This signals experience.

### 1. Off-by-One
- Loop bounds: `<` vs `<=`, `i` vs `i+1`
- `substring(start, end)` — end is **exclusive**
- Array last index: `arr.length - 1`, not `arr.length`
- Fence-post: N items have N-1 gaps

### 2. Null Handling
- Method returns null, caller doesn't check → `NullPointerException`
- Null in collection → surprise NPE when unboxing
- Boxed primitives: `Integer x = null; int y = x;` → NPE (auto-unboxing)
- `Optional.get()` without `isPresent()` check

### 3. Wrong Operator
- `==` vs `equals()` for objects (especially String)
- `&` vs `&&`, `|` vs `||` (bitwise vs short-circuit)
- `>` vs `>=`
- Integer division: `5 / 2 == 2` not `2.5` — need `5.0 / 2`
- Assignment `=` vs comparison `==` in conditions

### 4. Mutation During Iteration
- `for (X x : list) { list.remove(x); }` → `ConcurrentModificationException`
- Fix: `Iterator.remove()`, `removeIf()`, or collect-then-remove
- Same with `HashMap` — can't modify during `for-each`

### 5. Integer Overflow
- `int * int` overflows silently to negative
- Binary search: `(low + high) / 2` overflows → use `low + (high - low) / 2`
- Large array index calculations

### 6. Resource Leak
- Streams, connections, file handles not closed
- Fix: try-with-resources (`AutoCloseable`)
- Common in JDBC: `Connection`, `PreparedStatement`, `ResultSet`

### 7. Wrong Base Case
- Recursion returns wrong value at base
- Empty input not handled — returns null or throws
- Single-element input — boundary not checked

### 8. Check-Then-Act (Concurrency)
- `if (!map.containsKey(k)) map.put(k, v)` — racy
- Fix: `computeIfAbsent()` or `putIfAbsent()`
- Any "if (condition) { action }" on shared state is suspect

### 9. Visibility (Concurrency)
- One thread's writes never seen by another
- Missing `volatile` or `synchronized`
- Reading a field set by another thread without memory barrier

### 10. Wrong Default / Initialisation
- `int` defaults to `0`, `boolean` to `false`, object to `null`
- Bug: should have been initialised to `-1`, `Integer.MAX_VALUE`, or empty collection

### 11. Floating Point
- `0.1 + 0.2 != 0.3` — floating-point representation
- Fix: `Math.abs(a - b) < epsilon` or use `BigDecimal` for money

---

## The Dry-Run Discipline

The single most important debugging skill. Practice until smooth:

1. **Write column headers** for every variable that matters
2. **Read each line aloud** — what does it do in plain English?
3. **Update columns** as variables change — don't trust your head
4. **At branches** — voice the condition and which branch is taken
5. **At loops** — count first 2 and last 2 iterations explicitly
6. **At method calls** — voice what's passed in and what comes back

### What NOT to do
- ❌ Stare and try to spot the bug visually — trace instead
- ❌ Guess "I bet it's the loop" without confirming
- ❌ Fix without retracing the failing case to confirm the fix
- ❌ Fix without checking another test still passes
- ❌ Go silent for 30+ seconds — voice the trace

---

## Five Classic Concurrency Debug Snippets

### Snippet A: Singleton race
```java
public class Config {
    private static Config instance;
    public static Config getInstance() {
        if (instance == null) { instance = new Config(); }
        return instance;
    }
}
// Bug: not thread-safe. Fix: DCL with volatile, or enum singleton.
```

### Snippet B: Volatile counter
```java
private volatile int count;
public void increment() { count++; }
// Bug: ++ is not atomic even with volatile. Fix: AtomicInteger.
```

### Snippet C: HashMap cache
```java
private final Map<String, Data> map = new HashMap<>();
public Data get(String key) {
    if (!map.containsKey(key)) { map.put(key, load(key)); }
    return map.get(key);
}
// Bug: HashMap not thread-safe; check-then-act race. Fix: ConcurrentHashMap.computeIfAbsent.
```

### Snippet D: Producer missing sync
```java
public synchronized void put(Item i) { items.add(i); }
public Item take() {
    if (items.isEmpty()) return null;
    return items.remove(0);
}
// Bug: take() not synchronized. Fix: synchronize take(), or use BlockingQueue.
```

### Snippet E: Spurious wakeup
```java
public synchronized void waitForReady() {
    if (!ready) wait();
}
// Bug: should be while, not if (spurious wakeups). Plus missing InterruptedException handling.
```

---

## UMPIRE for Debugging (adapted)

| Step | Action | Time |
|------|--------|:----:|
| **U**nderstand | Read entire function. Voice what it does. State data structures used. | 90s |
| **M**ap | Read failing test. State expected vs actual. State the input. | 60s |
| **P**redict | Name 2–3 suspicious lines and why. | 30s |
| **I**nspect | Dry-run the failing case line by line. Voice variables. Stop when expected ≠ actual. | 5–8m |
| **R**epair | State fix in English → apply → re-trace failing case → trace another test. | 3–5m |
| **E**valuate | State root cause with technical term. Note if pattern exists elsewhere. | 60s |

---

## Can you answer these cold?

- [ ] Name 10 bug types from taxonomy — one sentence each
- [ ] Given Snippet C, name the bug type and fix in 15 seconds
- [ ] Dry-run a 30-line method aloud, tracking variables
- [ ] UMPIRE — all 6 steps with timing
- [ ] "Check-then-act race" — define it, give an example, give the fix

[← Back to Index](./00_INDEX.md)
