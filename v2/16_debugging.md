# 16 — Bug Taxonomy & Debugging

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## 🟢 Start Here — Debugging in Plain English

### What is debugging?

Debugging is **finding and fixing bugs** — mistakes in your code that cause wrong behaviour. It's like being a detective: your code is the crime scene, the bug is the culprit, and you need to figure out what went wrong.

### The #1 debugging skill: Don't guess — trace

The biggest mistake beginners make is **staring at the code and guessing** what's wrong. Instead, **trace through the code step by step**, tracking what each variable contains.

```java
// Bug: this method should return the sum, but it's returning wrong values
int sum(int[] nums) {
    int total = 0;
    for (int i = 1; i <= nums.length; i++) {   // 🐛 bug is here!
        total += nums[i];
    }
    return total;
}

// TRACE with input: nums = [10, 20, 30]
// i=1: total += nums[1] → total = 20     (skipped nums[0]! Should start at i=0)
// i=2: total += nums[2] → total = 50
// i=3: total += nums[3] → 💥 ArrayIndexOutOfBoundsException! (i should be < length, not <=)
// TWO bugs: starts at 1 instead of 0, and uses <= instead of <
```

### The five-step method

1. **Reproduce** — Can you make the bug happen reliably?
2. **Isolate** — What's the smallest input that causes it?
3. **Trace** — Walk through the code line by line with the failing input
4. **Find** — Where does the actual result first diverge from expected?
5. **Fix & verify** — Apply the fix, re-run with the failing input AND other inputs

### Most common bug types (learn to name them)

| Bug type | What it looks like | Example |
|----------|-------------------|---------|
| **Off-by-one** | Loop runs one too many or few times | `i <= length` instead of `i < length` |
| **Null pointer** | Something is `null` when you don't expect it | `user.getName()` when `user` is null |
| **Wrong operator** | Used wrong comparison or operation | `==` instead of `equals()` for Strings |
| **Race condition** | Two threads mess with the same data | Balance goes negative in multithreaded code |
| **Resource leak** | Forgot to close file/connection | File stays open → eventually run out of handles |

### The dry-run technique

When you get a code snippet in an interview:

1. **Write a table** with a column for each variable
2. **Step through each line** and update the columns
3. **At loops** — do the first 2 and last 2 iterations explicitly
4. **Say it out loud** — "i is now 3, total is now 50, the condition is true so we enter the loop"

This technique catches most bugs within 2-3 minutes.

> Key takeaway: Don't guess, trace. Walk through code line by line with actual values. Name the bug type (off-by-one, null pointer, race condition). Fix it, then verify with multiple inputs.

---

## 📚 Study Material

### 1. Systematic Debugging Methodology

**The #1 mistake:** Guessing. The #1 skill: **systematic tracing**.

```
Step 1: REPRODUCE — Can you reliably trigger the bug?
        If intermittent → likely concurrency or timing-related

Step 2: ISOLATE — What's the smallest input that triggers it?
        Binary search: remove half the input, still broken? Narrow down.

Step 3: TRACE — Follow the data flow line by line
        Don't trust your mental model — write down variable values

Step 4: COMPARE — Expected vs actual at each step
        The first point where they diverge is your bug

Step 5: FIX — Apply the fix, then verify:
        ✅ Failing test now passes
        ✅ Other tests still pass
        ✅ Fix addresses root cause, not just the symptom
```

### 2. Production Debugging Tools

```bash
# Thread dump — see what every thread is doing RIGHT NOW
jstack <pid>
# Look for: BLOCKED threads (deadlock), threads stuck in I/O, thread pool exhaustion

# Heap dump — snapshot of all objects in memory
jmap -dump:format=b,file=heap.hprof <pid>
# Or: -XX:+HeapDumpOnOutOfMemoryError (automatic on OOM)
# Analyse with: Eclipse MAT, VisualVM
# Look for: largest retained objects, leak suspects, dominator tree

# Java Flight Recorder — low-overhead continuous profiling
jcmd <pid> JFR.start duration=60s filename=recording.jfr
# Analyse with: JDK Mission Control
# Shows: CPU hotspots, allocations, GC pauses, thread activity, I/O wait

# GC logs — understand memory behaviour
-Xlog:gc*:file=gc.log:time,uptime,level,tags
# Look for: frequency of GC, pause times, heap growth trends
# Full GC too frequent → memory leak or heap too small
```

**Reading a thread dump:**
```
"http-nio-8080-exec-1" #42 daemon prio=5
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.citi.service.TradeService.execute(TradeService.java:45)
        - waiting to lock <0x00000007e0123456> (a com.citi.model.Account)
        - locked <0x00000007e0654321> (a com.citi.model.Account)

"http-nio-8080-exec-2" #43 daemon prio=5
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.citi.service.TradeService.execute(TradeService.java:45)
        - waiting to lock <0x00000007e0654321>     ← wants what exec-1 holds
        - locked <0x00000007e0123456>               ← holds what exec-1 wants

// DEADLOCK: exec-1 holds A, wants B. exec-2 holds B, wants A.
```

### 3. Log Analysis Patterns

```java
// STRUCTURED LOGGING — makes logs searchable
log.info("Trade executed: tradeId={}, symbol={}, quantity={}, latencyMs={}",
         trade.getId(), trade.getSymbol(), trade.getQuantity(), elapsed);
// In JSON log format: { "tradeId": "T001", "symbol": "AAPL", ... }
// Searchable in ELK/Splunk: tradeId=T001 AND symbol=AAPL

// MDC — attach context to all log lines in a request
MDC.put("requestId", UUID.randomUUID().toString());
MDC.put("userId", user.getId());
try {
    // ALL log lines in this request automatically include requestId and userId
    service.processOrder(order);  // logs inside processOrder() also get the MDC context
} finally {
    MDC.clear();  // MUST clean up (thread pool reuse)
}

// Log pattern: %d{ISO8601} [%X{requestId}] %-5level %logger{36} - %msg%n
// Output: 2024-03-15T10:23:45 [abc-123] INFO  TradeService - Trade executed...
```

### 4. Common Debugging Scenarios

**Scenario A: API returns 500 but no error in logs**
```
Checklist:
1. Check if @ControllerAdvice is swallowing the exception
2. Check if @Async method threw (silently swallowed by default)
3. Check Spring Security filter chain — may reject before reaching controller
4. Check if reverse proxy (nginx) is returning its own 500
5. Increase log level: logging.level.org.springframework=DEBUG
```

**Scenario B: Memory keeps growing (suspected leak)**
```
1. Enable GC logging → is Old Gen growing between Full GCs?
2. Take heap dumps at T=0 and T+30min
3. Compare in Eclipse MAT → "Leak Suspects" report
4. Common culprits:
   - Static collections that grow unbounded
   - Listeners/callbacks never unregistered
   - ThreadLocal not cleaned up in thread pools
   - Hibernate session cache (first-level cache) in long transactions
```

**Scenario C: Intermittent failures under load**
```
1. Thread dump during failure → look for BLOCKED threads
2. Check connection pool exhaustion (HikariCP max-pool-size)
3. Check thread pool exhaustion (Tomcat max-threads)
4. Check for lock contention (thread dump shows synchronized waits)
5. Check for race conditions (review shared mutable state)
```

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
