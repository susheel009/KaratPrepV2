# 16 — Bug Taxonomy & Debugging

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/16_debugging_doubts.md) · [← Prev: 15 Kafka](./15_kafka.md) · [Next: 17 Debugging Problems →](./17_debugging_problems.md)
>
> **Priority:** 🔴 Critical · **Related topics:** [17 Debugging Problems (practice set)](./17_debugging_problems.md) · [01 Concurrency](./01_concurrency.md) · [02 Collections](./02_collections.md) · [07 JVM](./07_jvm.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — most bugs aren't where you think
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — finding two bugs in one method
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — dry-run discipline
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 The systematic method](#41-the-systematic-method-five-steps)
   - [4.2 Bug taxonomy — 11 named types](#42-bug-taxonomy--11-named-types)
   - [4.3 The dry-run discipline](#43-the-dry-run-discipline)
   - [4.4 Production tools](#44-production-debugging-tools)
   - [4.5 Reading a thread dump](#45-reading-a-thread-dump)
   - [4.6 Log analysis patterns](#46-log-analysis-patterns)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 UMPIRE for debugging](#51-umpire-for-debugging-interview-cadence)
   - [5.2 Common production scenarios](#52-common-production-scenarios)
   - [5.3 Five classic concurrency snippets](#53-five-classic-concurrency-snippets)
   - [5.4 Pattern recognition shortcuts](#54-pattern-recognition-shortcuts)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§4.2 Bug Taxonomy](#42-bug-taxonomy--11-named-types) and [§5.3 Five Classic Snippets](#53-five-classic-concurrency-snippets).

---

## 1. The Problem (story-style, no code)

A detective walks into a house where there's been a robbery. They don't stand in the doorway and *guess* who did it. They don't shout "It must have been the gardener!" before checking the kitchen. They walk the scene methodically — door, window, drawer, footprint — until evidence forces a conclusion.

Most engineers debug like the bad detective. They stare at code, squint, and *guess* "I bet the loop is wrong." Then they tweak something, hope, run, fail, tweak something else. The cycle stretches into hours.

The professionals do something quietly different. They have:

1. **A method.** Reproduce → isolate → trace → find divergence → fix → verify. Same five steps every time, regardless of bug.
2. **A vocabulary.** 11 named bug types. Naming the type ("that's a check-then-act race") cuts your search space by 90 %.
3. **A discipline.** When tracing, they write down every variable. They don't trust their head. They voice the trace aloud. The mental act of saying "i is 3, total is 50, the condition is true" catches bugs faster than the most concentrated stare.

This file is the toolkit. By the end of it you should be able to:

- Read a 30-line method, dry-run it aloud, name the bug type, and propose the fix — under 5 minutes.
- Look at a thread dump and spot a deadlock or thread starvation.
- Know which JVM tool to reach for (jstack, jmap, JFR) for which symptom.

> The companion file [17 Debugging Problems](./17_debugging_problems.md) is 20 buggy snippets to practise on. Read this file first, then drill on 17.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

You're handed this method:

```
function sum(nums):
    total = 0
    for i = 1 to nums.length (inclusive):
        total += nums[i]
    return total
```

The interviewer says: "this should sum the array, but it's wrong. Tell me what's broken."

The amateur thinks: "Hmm, the loop runs through the array, returns total. Looks fine?"

The professional dry-runs it with input `[10, 20, 30]`:

| Iteration | i | nums[i] | total |
|---|---:|---:|---:|
| start | — | — | 0 |
| 1 | 1 | 20 | 20 |
| 2 | 2 | 30 | 50 |
| 3 | 3 | **💥 ArrayIndexOutOfBounds — index 3 doesn't exist** | — |

Two bugs uncovered, named:

1. **Off-by-one — wrong start.** Loop starts at `i=1`, skipping `nums[0]` (the value 10). The first lost sum.
2. **Off-by-one — wrong end.** `i ≤ nums.length` walks past the last index (which is `length - 1`). The crash.

Fix: `for i = 0; i < nums.length; i += 1`.

Two takeaways the walkthrough makes obvious:

1. **Don't trust your reading of the code — trust the trace.** The bug stayed hidden until we wrote the values down. The code "looked fine" because that's how off-by-ones look.
2. **Naming the bug type signals expertise.** "Off-by-one" is a one-second sentence; without the name you'd describe the same thing in three sentences and still sound vague.

Now actual code, plus the discipline to drill it.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
// You are given this. Find the bug.
public class Average {
    public static double average(int[] xs) {
        int sum = 0;
        for (int i = 0; i <= xs.length; i++) {       // ⚠ <= — off-by-one suspect 1
            sum += xs[i];
        }
        return sum / xs.length;                       // ⚠ integer division — suspect 2
    }
    public static void main(String[] args) {
        System.out.println(average(new int[]{1, 2, 3, 4}));   // expect 2.5
    }
}
```

Dry-run with `xs = [1, 2, 3, 4]`:

| i | xs[i] | sum | running |
|---:|---:|---:|---|
| 0 | 1 | 1 | OK |
| 1 | 2 | 3 | OK |
| 2 | 3 | 6 | OK |
| 3 | 4 | 10 | OK |
| 4 | **AIOOBE** | — | crash, `i < length` should be the bound |

Bugs found:
1. **Off-by-one** — `i <= xs.length` walks past the last valid index. Fix: `i < xs.length`.
2. **Integer division** — `sum / xs.length` is `int / int` → integer division. With `sum=10, length=4` it returns 2 (truncated), not 2.5. Fix: `(double) sum / xs.length` or `sum / (double) xs.length`.

```java
// Fixed
return (double) sum / xs.length;
for (int i = 0; i < xs.length; i++) sum += xs[i];
```

What just happened: 7 lines of code, 2 named bugs, 30 seconds of disciplined dry-run. **The discipline is the skill.** Once you have it, most "tricky" interview snippets become a 2-minute exercise.

---

## 4. Build Up — Practical Patterns

### 4.1 The systematic method (five steps)

```
1. REPRODUCE — Can you reliably trigger the bug?
   Intermittent? → likely concurrency / timing / state-dependent.
   Always? → easier; just trace.

2. ISOLATE — What's the smallest input that triggers it?
   Binary-search the input. Halve it; still broken? Halve again.
   "It's broken on this 1-element list, not on 0-element" — narrowed dramatically.

3. TRACE — Walk through the data flow line by line.
   Don't trust your mental model. Write down every variable.
   Voice the trace aloud — the act of speaking catches things the eyes miss.

4. COMPARE — Expected vs actual at each step.
   The first point where they diverge is your bug.

5. FIX & VERIFY — Apply the fix, retrace the failing case, run other tests.
   ✅ Failing test now passes.
   ✅ Other tests still pass.
   ✅ Fix addresses the root cause, not just the symptom.
```

### 4.2 Bug taxonomy — 11 named types

When you spot a bug in an interview, **name the type**. It signals experience and shortens explanations.

#### 1. Off-by-one
- Loop bounds: `<` vs `<=`, `i` vs `i+1`.
- `String.substring(start, end)` — end is **exclusive**.
- Last index: `arr.length - 1`, not `arr.length`.
- Fence-post: N items have N-1 gaps.

#### 2. Null handling
- Method returns `null`; caller doesn't check → `NullPointerException`.
- `null` in a collection → surprise NPE on iteration / unboxing.
- Auto-unboxing: `Integer x = null; int y = x;` → NPE.
- `Optional.get()` without `isPresent` check.

#### 3. Wrong operator
- `==` vs `.equals` for objects (especially `String` — see [05](./05_strings.md#42--vs-equals--reference-vs-content)).
- `&` vs `&&`, `|` vs `||` (bitwise vs short-circuit).
- `>` vs `>=`.
- Integer division: `5 / 2 == 2`. Need `5.0 / 2`.
- Assignment `=` vs comparison `==` in conditions.

#### 4. Mutation during iteration
- `for (X x : list) { list.remove(x); }` → `ConcurrentModificationException`.
- Fix: `Iterator.remove()`, `removeIf()`, or collect-then-remove. See [02 §4.5](./02_collections.md#45-iteration-safety--concurrentmodificationexception).

#### 5. Integer overflow
- `int * int` overflows silently to a negative number.
- Binary search: `(low + high) / 2` overflows → use `low + (high - low) / 2`.
- Also relevant for time: `currentTimeMillis` differences.

#### 6. Resource leak
- Streams, JDBC connections, file handles not closed.
- Fix: try-with-resources (`AutoCloseable`). See [04 §4.3](./04_exceptions.md#43-try-with-resources-and-autocloseable-java-7).

#### 7. Wrong base case (recursion)
- Empty input not handled — returns null, throws, or recurses forever.
- Single-element input — boundary not checked.
- Recursion returns wrong value at the base.

#### 8. Check-then-act (concurrency)
- `if (!map.containsKey(k)) map.put(k, v)` — racy.
- Fix: `computeIfAbsent` / `putIfAbsent` (atomic). See [02 §5.3](./02_collections.md#53-atomic-compound-operations-on-concurrenthashmap).
- Any "if (condition) { do something }" on shared state is suspect.

#### 9. Visibility (concurrency)
- One thread's writes are never seen by another (cached in CPU register / cache).
- Missing `volatile` or `synchronized`.
- See [01 §4.6](./01_concurrency.md#46-volatile--visibility-not-atomicity).

#### 10. Wrong default / initialisation
- `int` defaults to `0`; `boolean` to `false`; objects to `null`.
- Bug: should have been `-1` (sentinel), `Integer.MAX_VALUE` (for "find min"), or empty collection (for "no items yet").

#### 11. Floating point
- `0.1 + 0.2 != 0.3` — IEEE 754 representation.
- Fix: `Math.abs(a - b) < epsilon` for comparison; `BigDecimal` for money.

### 4.3 The dry-run discipline

The single most important debugging skill. Practise until smooth:

1. **Write a column for each variable that matters.** Pen-and-paper, or whiteboard, or a comment table in the IDE.
2. **Read each line aloud** — what does it do, in plain English?
3. **Update the columns** as variables change. Don't trust your head.
4. **At every branch** — voice the condition and which branch is taken.
5. **At every loop** — count first 2 and last 2 iterations explicitly. Skip the middle, but verify the boundary.
6. **At every method call** — voice what's passed in and what comes back.

What NOT to do:

- ❌ Stare at the code looking for the bug visually — trace.
- ❌ Guess "I bet the loop is wrong" without confirming.
- ❌ Fix without retracing the failing input to confirm.
- ❌ Fix without checking another test still passes.
- ❌ Go silent for 30+ seconds — voice the trace.

> The interviewer scores not just whether you find the bug but **how**. Visible discipline beats lucky guesses every time.

### 4.4 Production debugging tools

```bash
# Thread dump — what every thread is doing right now
jstack <pid>
# Look for: BLOCKED threads (deadlock), threads stuck in I/O, thread pool exhaustion
# Or: jcmd <pid> Thread.print

# Heap dump — snapshot of all objects in memory
jcmd <pid> GC.heap_dump /tmp/heap.hprof
# Analyse with: Eclipse MAT (dominator tree), VisualVM
# -XX:+HeapDumpOnOutOfMemoryError takes one automatically on OOM

# Java Flight Recorder — low-overhead continuous profiling
jcmd <pid> JFR.start duration=60s filename=recording.jfr
# Analyse with: JDK Mission Control
# Shows: CPU hotspots, allocations, GC pauses, thread activity, I/O wait

# Native memory tracking
jcmd <pid> VM.native_memory summary

# GC logs — understand memory behaviour
-Xlog:gc*:file=gc.log:time,uptime,level,tags
# Look for: GC frequency, pause times, heap-after-GC trend
# Old Gen growing between Full GCs → memory leak.
```

See [07 JVM §5.7](./07_jvm.md#57-diagnostics-jmap-jstack-jfr-mat) for fuller coverage.

### 4.5 Reading a thread dump

A deadlock as it appears in a real thread dump:

```
"http-nio-8080-exec-1" #42 daemon prio=5
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.service.TradeService.execute(TradeService.java:45)
        - waiting to lock <0x00000007e0123456> (a com.example.model.Account)
        - locked          <0x00000007e0654321> (a com.example.model.Account)

"http-nio-8080-exec-2" #43 daemon prio=5
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.service.TradeService.execute(TradeService.java:45)
        - waiting to lock <0x00000007e0654321>     ← what exec-1 holds
        - locked          <0x00000007e0123456>     ← what exec-1 wants

DEADLOCK: exec-1 holds A, wants B; exec-2 holds B, wants A.
```

Other patterns to recognise:

| Pattern | What it tells you |
|---|---|
| Many threads in `BLOCKED` on the same monitor | Lock contention — single-threaded bottleneck. |
| Threads in `RUNNABLE` parked on `Socket.read` | Threads blocked on I/O; thread pool exhaustion likely. |
| Threads in `WAITING` on `LockSupport.park` | Awaiting semaphore / condition / future / lock acquisition. |
| All worker threads `BLOCKED` on the same DB pool | Connection pool exhaustion. |
| Hundreds of `http-nio-*` threads | Tomcat saturated; check downstream latency. |

### 4.6 Log analysis patterns

```java
// STRUCTURED LOGGING — searchable
log.info("Trade executed: tradeId={} symbol={} qty={} latencyMs={}",
         trade.id(), trade.symbol(), trade.qty(), elapsed);
// In Splunk/ELK: tradeId=T001 AND symbol=AAPL → exact filter.

// MDC — attach context to every log line in a request
MDC.put("requestId", UUID.randomUUID().toString());
MDC.put("userId", user.id());
try {
    service.process(order);                // logs deep inside also include requestId/userId
} finally {
    MDC.clear();                           // ⚠ MUST clean up — thread pool reuse leaks context
}
// Logback pattern: %d{ISO8601} [%X{requestId}] %-5level %logger{36} - %msg%n
// Output:           2026-05-01T10:23:45 [abc-123] INFO  TradeService - Trade executed: tradeId=...
```

> ⚠ **Always `MDC.clear()` in a finally block** in pooled environments. If you don't, a request's context will leak to the next request handled on the same thread — privacy and audit issues. Same trap as `ThreadLocal`. See [01 §5.8](./01_concurrency.md#58-threadlocal--per-thread-storage-mind-the-leak).

---

## 5. Going Deep — Interview-Level Material

### 5.1 UMPIRE for debugging (interview cadence)

A 6-step cadence with explicit timing — designed for a 20-minute screen with a buggy snippet:

| Step | Action | Time |
|---|---|:---:|
| **U**nderstand | Read the entire function. Voice what it's supposed to do. State the data structures used. | ~90 s |
| **M**ap | Read the failing test or expected behaviour. State expected vs actual. State the input. | ~60 s |
| **P**redict | Name 2–3 suspicious lines and why. *Predict before tracing*. | ~30 s |
| **I**nspect | Dry-run the failing case line by line. Voice variables. Stop at the first divergence. | 5–8 min |
| **R**epair | State the fix in English. Apply it. Re-trace the failing case to confirm. Trace another case to make sure you didn't break it. | 3–5 min |
| **E**valuate | State the root cause using the technical bug-type name. Note if the same pattern exists elsewhere in the snippet. | ~60 s |

The discipline matters because under pressure, people abandon method. The cadence gives you a script to fall back on when nerves kick in.

### 5.2 Common production scenarios

**Scenario A — "API returns 500 but no error in logs"**

Checklist:
1. Is `@RestControllerAdvice` swallowing the exception? (Returning a generic 500 without logging.)
2. Is an `@Async` method throwing? (Default executor silently swallows void-method exceptions — see [12 §4.6](./12_spring_boot.md#46-async--background-execution).)
3. Is Spring Security rejecting before the controller even runs?
4. Is a reverse proxy (nginx, ALB) returning its own 500 — your app is fine?
5. Increase log level: `logging.level.org.springframework=DEBUG`.

**Scenario B — "memory keeps growing"**

1. Enable GC logs. Is Old Gen growing between Full GCs? Yes → real leak.
2. Take heap dumps at T=0 and T+30 min.
3. Open in Eclipse MAT → "Leak Suspects" report → dominator tree.
4. Common culprits:
   - Static collection that grows unbounded (cache without eviction).
   - Listeners / callbacks never unregistered.
   - `ThreadLocal` not cleaned up in pooled threads.
   - Hibernate session cache (first-level cache) in long-running transaction.
   - Class loader leak on app-server redeploy — see [07 §5.3](./07_jvm.md#53-class-loader-delegation-and-leaks).

**Scenario C — "intermittent failures under load"**

1. Thread dump *during* the failure — look for `BLOCKED` threads.
2. Connection pool exhaustion (HikariCP `maximum-pool-size` default is 10).
3. Tomcat thread pool exhaustion (`server.tomcat.threads.max`).
4. Lock contention — many threads `BLOCKED` on the same monitor.
5. Race condition on shared mutable state — review `synchronized` / `volatile` / `Atomic*` use.

### 5.3 Five classic concurrency snippets

You'll see these or close cousins repeatedly. Name and fix from memory:

#### Snippet A — Singleton race
```java
public class Config {
    private static Config instance;
    public static Config getInstance() {
        if (instance == null) { instance = new Config(); }
        return instance;
    }
}
// Bug: not thread-safe. Two threads can both see null and both create.
// Fix: enum singleton, or DCL with volatile (see [01 §5.1]).
```

#### Snippet B — `volatile` counter
```java
private volatile int count;
public void increment() { count++; }
// Bug: ++ is read-add-write; volatile gives visibility but NOT atomicity.
// Fix: AtomicInteger or LongAdder. See [01 §4.6 + §4.7].
```

#### Snippet C — `HashMap` cache (race + non-thread-safe map)
```java
private final Map<String, Data> map = new HashMap<>();
public Data get(String key) {
    if (!map.containsKey(key)) { map.put(key, load(key)); }
    return map.get(key);
}
// Bug: HashMap not thread-safe (resize during concurrent put = corruption);
//      check-then-act race even with ConcurrentHashMap.
// Fix: ConcurrentHashMap.computeIfAbsent.
```

#### Snippet D — Producer missing sync
```java
public synchronized void put(Item i) { items.add(i); }
public Item take() {
    if (items.isEmpty()) return null;
    return items.remove(0);
}
// Bug: take() not synchronized — race on items.
// Fix: synchronize take(); or use BlockingQueue (see [01 §5.7]).
```

#### Snippet E — Spurious wakeup
```java
public synchronized void waitForReady() {
    if (!ready) wait();
}
// Bug: should be `while (!ready)`, not `if`. Spurious wakeups can return from wait()
// without anyone calling notify(). Plus missing InterruptedException handling.
// See [01 §5.4].
```

### 5.4 Pattern recognition shortcuts

When you see this in code review, suspect this:

| Pattern | Suspect |
|---|---|
| `for (X x : list) { list.remove(x); }` | CME — mutation during iteration |
| `if (!map.containsKey(k)) map.put(k, v)` | Check-then-act race |
| `volatile int x; x++;` | Visibility ≠ atomicity — broken counter |
| `if (instance == null) instance = new …;` | Singleton race; need DCL + volatile |
| `(low + high) / 2` | Integer overflow on large arrays |
| `s == "literal"` | Reference vs value equality (use `.equals`) |
| `for (i = 1; i <= arr.length; …)` | Off-by-one on both ends |
| `5 / 2` returning 2.0 expected | Integer division — cast to double |
| `try { … } catch (Exception e) {}` | Swallowed exception |
| `@Transactional` + `this.method()` | Self-invocation bypass |
| Optional.get() without `isPresent` | NoSuchElementException waiting |
| ThreadLocal with `set` and no `remove` | Leak in pooled environments |

---

## 6. Memory Aids

### Decision tree: "I see this — what's likely wrong?"

```
Crash with NullPointerException?
└── Some method returned null (or value got auto-unboxed) — find it via stack trace.

ConcurrentModificationException?
└── Mutation during iteration — Iterator.remove() / removeIf() / collect-then-remove.

Wrong total / wrong count?
├── Off-by-one on a loop bound — dry-run boundary iterations.
├── Integer division dropping decimals — cast operands to double.
├── Integer overflow — use long, or use low + (high - low)/2 idiom.
└── Concurrency: missed update — check-then-act or volatile-without-atomic.

Intermittent failure?
├── Thread dump for BLOCKED / contention.
├── Race condition on shared mutable state.
└── Resource exhaustion (DB pool, thread pool).

Memory growing forever?
├── Heap dump → MAT → dominator tree.
├── Static collection unbounded growth.
├── ThreadLocal not cleaned in pool.
└── Listeners / callbacks unregistered.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "What's wrong with this code?" | Dry-run, don't guess | "Let me trace it with a sample input." |
| "Off-by-one?" | Loop bound or substring end | "End is exclusive in substring. <= vs < on loops." |
| "ConcurrentModificationException?" | Mutation during iteration | "Iterator.remove() or removeIf()." |
| "x++ on volatile?" | Read-add-write, not atomic | "Visibility yes, atomicity no. Use AtomicInteger." |
| "Why deadlock?" | Two threads, two locks, opposite order | "Always acquire locks in the same global order." |
| "How to debug high memory?" | Heap dump → MAT → dominator tree | "Look for the largest retainers and trace to GC roots." |
| "Why intermittent?" | Concurrency / timing / state | "Reproduce reliably first; intermittent = state-dependent." |
| "Should I fix the symptom?" | No — find root cause | "Symptom fix masks the bug; same pattern shows up elsewhere later." |

### Three anchor pictures

1. **Trace, don't guess.** Write down every variable; voice every step.
2. **Name the bug type.** "Off-by-one", "check-then-act race", "visibility bug" — vocabulary cuts search time.
3. **Fix the root cause.** Symptom fixes return to bite later, often as worse bugs.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: What's the systematic debugging method?
**A:** Reproduce → isolate → trace → compare → fix & verify. Same five steps every time. Don't skip any of them.

### Q2: What's the dry-run discipline?
**A:** Trace through code line by line, writing down each variable's value, voicing it aloud. Don't trust your head. Verify boundary iterations on loops. Voice the condition at each branch.

### Q3: List six common bug types.
**A:** Off-by-one (loop bounds, substring), null handling, wrong operator (`==` vs `.equals`, integer division), mutation during iteration (CME), integer overflow, resource leak, check-then-act race, visibility, wrong base case, wrong default, floating point.

### Q4: What's a check-then-act race?
**A:** A check on shared state followed by an action that depends on the check, with no synchronisation between them. Another thread can change the state between check and act, invalidating the assumption. Fix: atomic compound operation (`computeIfAbsent`, `putIfAbsent`).

### Q5: Why is `volatile int x; x++;` broken?
**A:** `++` is read-add-write — three operations. `volatile` makes each read and each write visible across threads, but doesn't make the trio atomic. Another thread can read between your read and your write, causing a lost update. Use `AtomicInteger.incrementAndGet()`.

### Q6: How do you read a thread dump for deadlock?
**A:** Look for two or more threads in `BLOCKED` state where each is `waiting to lock` an object the other has `locked`. The cycle of "waiting to lock X, locked Y" → "waiting to lock Y, locked X" is the deadlock signature.

### Q7: What's MDC and what's the trap?
**A:** Mapped Diagnostic Context — per-thread map of key/value pairs added to every log line. Trap: in pooled threads, a request's MDC values bleed to the next request unless you `MDC.clear()` in a `finally` block.

### Q8: How would you debug "API returns 500 but no error in logs"?
**A:** (1) Check `@RestControllerAdvice` for swallowing. (2) Check `@Async` void methods (silently swallow exceptions by default). (3) Check Spring Security — may reject before controller. (4) Check reverse proxy — may be its own 500. (5) Increase Spring log level to DEBUG.

### Q9: How do you find a memory leak?
**A:** Enable GC logs; verify Old Gen grows. Take heap dumps at T=0 and T+30min; compare in Eclipse MAT. Use the dominator tree to find the largest retainers; walk references back to a GC root. Common culprits: static unbounded collections, ThreadLocal in pools, listener leaks, class-loader leaks on redeploy.

### Q10: What does a thread dump tell you about thread-pool exhaustion?
**A:** Many threads named after the pool (`http-nio-*`, `task-*`) all `BLOCKED` on a single resource (DB pool, lock, downstream HTTP). When the pool is at max and all threads are blocked, no new requests are processed.

### Q11: Why is mutation during iteration a problem?
**A:** Fail-fast iterators detect structural change via a `modCount` field and throw `ConcurrentModificationException`. Fix: `Iterator.remove()` (modifies via the iterator), `Collection.removeIf(predicate)` (Java 8+), or collect targets then remove afterward.

### Q12: How does the `(low + high) / 2` bug manifest?
**A:** When `low + high` overflows `int.MAX_VALUE`, the result wraps to negative. Mid becomes negative; binary search returns wrong index or crashes. Fix: `low + (high - low) / 2`.

### Q13: Why use `BigDecimal` instead of `double` for money?
**A:** Floating point can't represent decimal fractions exactly. `0.1 + 0.2 != 0.3` in IEEE 754. Cumulative rounding errors corrupt totals over time. `BigDecimal` is exact decimal arithmetic.

### Q14: What's the difference between a symptom fix and a root-cause fix?
**A:** A symptom fix masks the visible failure (catch and ignore, retry, restart). A root-cause fix corrects the underlying flaw. Symptom fixes save time short-term but the bug appears elsewhere later, often more dangerously.

### Q15: How would you find why a `@Cacheable` annotation is being ignored?
**A:** Most likely: self-invocation. The `@Cacheable` proxy only intercepts external calls; `this.cachedMethod()` bypasses it. Fix: inject self via `@Lazy`, use `CacheManager` programmatically, or restructure.

### Q16: What's UMPIRE and when use it?
**A:** A 6-step interview cadence: Understand → Map → Predict → Inspect → Repair → Evaluate, with explicit timing. Use it under pressure to keep your reasoning structured when you're tempted to guess.

---

### Key Code/Diagnostic Patterns

**Heap dump on OOM**
```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/heap.hprof
```

**Thread dump (live)**
```bash
jcmd <pid> Thread.print > threads.txt
```

**MDC pattern with finally**
```java
try { MDC.put("requestId", id); doWork(); }
finally { MDC.clear(); }
```

**Dry-run header for tracing**
```
| iter | i | total | nums[i] |
|------|---|-------|---------|
|  0   | 0 |   0   |   10    |
…
```

---

## 8. Self-Test

**Easy**
- [ ] Name 6 of the 11 bug types.
- [ ] What's the dry-run discipline?
- [ ] Why is `volatile int x; x++;` broken?
- [ ] What does `ConcurrentModificationException` indicate?

**Medium**
- [ ] Walk through a 30-line method aloud, tracing variables.
- [ ] Read a sample thread dump and identify a deadlock.
- [ ] Why does `(low + high) / 2` cause bugs in binary search?
- [ ] How would you handle a "memory keeps growing" report in production?

**Hard**
- [ ] Apply UMPIRE to a snippet you've never seen — describe each step's deliverable.
- [ ] You see thousands of `BLOCKED` HTTP threads. Walk through the diagnosis.
- [ ] Why is a symptom fix sometimes worse than no fix? Give a concrete example.
- [ ] How does a class-loader leak manifest in production, and how do you confirm it?
- [ ] Give the names of all five classic concurrency snippets and the bug in each.

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Dry-run** | Tracing through code by hand, recording every variable's value at each line. |
| **Off-by-one** | A loop or index bound that's wrong by one (start at 1 instead of 0, etc.). |
| **NPE** | NullPointerException — dereferencing a null. |
| **CME** | ConcurrentModificationException — fail-fast iterator complaint about structural mutation. |
| **Check-then-act race** | A check on shared state followed by a dependent action with no synchronisation between. |
| **Visibility (concurrency)** | Whether one thread's writes are seen by another. Fixed by `volatile` / `synchronized`. |
| **Atomicity** | Whether a multi-step operation is indivisible. Fixed by `synchronized` / `Atomic*` / atomic compound ops. |
| **Stack trace** | The list of method calls at the point of an exception. |
| **Thread dump** | Snapshot of every thread's current stack and state. |
| **Heap dump** | Snapshot of every object on the heap, for memory analysis. |
| **JFR** | Java Flight Recorder — low-overhead continuous profiling. |
| **MAT** | Eclipse Memory Analyzer Tool — opens heap dumps, finds leak suspects. |
| **MDC** | Mapped Diagnostic Context — per-thread log context. |
| **Symptom fix** | A fix that hides the visible failure without addressing the cause. |
| **Root-cause fix** | A fix that corrects the underlying flaw. |
| **UMPIRE** | A 6-step cadence (Understand, Map, Predict, Inspect, Repair, Evaluate) for debugging in interviews. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/16_debugging_doubts.md) · [← Prev: 15 Kafka](./15_kafka.md) · [Next: 17 Debugging Problems →](./17_debugging_problems.md)

[↑ Back to top](#16--bug-taxonomy--debugging)
