# 07 — JVM Internals

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

---

## 🟢 Start Here — JVM in Plain English

### What is the JVM?

The **Java Virtual Machine** is the engine that runs your Java code. You write `.java` files → the compiler turns them into `.class` files (bytecode) → the JVM reads and executes that bytecode.

**Why "virtual"?** Because it pretends to be a computer. Your Java code runs the same on Windows, Mac, or Linux — the JVM handles the differences. "Write once, run anywhere."

### Memory — where things live

When your program runs, the JVM uses two main memory areas:

**Stack** = your desk (small, organised, per-person)
- Stores local variables and method calls
- Each thread (worker) has its own stack
- When a method finishes, its stuff is automatically cleaned up

**Heap** = the warehouse (big, shared)
- Stores objects (anything created with `new`)
- Shared by ALL threads
- Cleaned up by the **Garbage Collector**

```java
public void processOrder() {
    int quantity = 10;                    // quantity → lives on the STACK
    Order order = new Order("AAPL", 10);  // reference 'order' → STACK
                                          // the Order object itself → HEAP
}
// When this method returns, 'quantity' and the reference are gone from the stack.
// The Order object on the heap will be cleaned up by GC when nothing references it.
```

### Garbage Collection — automatic cleanup

In some languages (C, C++), you have to manually free memory. Java does it for you with **Garbage Collection (GC)**.

Think of GC as a **janitor** who walks around the warehouse. If an object isn't being used by anyone anymore (nothing points to it), the janitor throws it away.

**Generational GC:** Most objects are short-lived (temporary variables, intermediate results). Java exploits this:
- **Young generation** — new objects go here. Cleaned frequently and cheaply.
- **Old generation** — long-lived objects get promoted here. Cleaned less often, more expensive.

It's like sorting mail: most is junk (young gen, toss quickly), some is important and goes in the filing cabinet (old gen, review less often).

### JIT Compilation — Java gets faster over time

Java starts by **interpreting** your code (slow but immediate). As it runs, it notices which methods are called frequently ("hot" methods) and **compiles them to native machine code** (fast, like C). This is called **Just-In-Time compilation**.

> Key takeaways: JVM runs your bytecode on any OS. Stack = per-thread small memory for local vars. Heap = shared memory for objects. GC = automatic cleanup. JIT = Java gets faster the longer it runs.

---

## 📚 Study Material

### 1. JVM Memory Layout

```
┌─────────────────────────────────────────────────────────┐
│                      JVM PROCESS                         │
│                                                          │
│  ┌──────────────── HEAP (-Xms/-Xmx) ──────────────────┐│
│  │  Young Generation              Old Generation        ││
│  │  ┌──────┬────┬────┐           ┌──────────────────┐  ││
│  │  │ Eden │ S0 │ S1 │           │  Tenured Space    │  ││
│  │  │      │    │    │           │  (long-lived)     │  ││
│  │  └──────┴────┴────┘           └──────────────────┘  ││
│  │  new objects here              promoted from Young   ││
│  └──────────────────────────────────────────────────────┘│
│                                                          │
│  ┌─── Metaspace (native memory, not heap) ────────────┐ │
│  │  Class metadata, method bytecode, constant pool     │ │
│  │  -XX:MaxMetaspaceSize (unbounded by default)        │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─── Thread Stacks (-Xss per thread) ────────────────┐ │
│  │  Thread 1: [Frame][Frame][Frame]                    │ │
│  │  Thread 2: [Frame][Frame]         (each ~512KB-1MB) │ │
│  │  Thread N: [Frame][Frame][Frame][Frame]             │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─── Other ──────────────────────────────────────────┐ │
│  │  Code Cache (JIT-compiled native code)              │ │
│  │  Direct ByteBuffers (NIO)                           │ │
│  │  Native memory (JNI)                                │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Heap vs Stack:**

| Aspect | Stack | Heap |
|--------|-------|------|
| Stores | Method frames, local primitives, references | Objects, arrays |
| Per-thread? | ✅ Each thread has its own | Shared across all threads |
| Size | Small (-Xss, default ~512KB-1MB) | Large (-Xms/-Xmx) |
| Speed | Very fast (LIFO push/pop) | Slower (GC-managed allocation) |
| Overflow | `StackOverflowError` (deep recursion) | `OutOfMemoryError` |

```java
public void transfer(Account from, Account to, int amount) {
    // 'from', 'to' → references on the stack (point to heap objects)
    // 'amount' → primitive on the stack
    // Account objects → on the heap
    BigDecimal bd = new BigDecimal(amount);  // 'bd' reference on stack, BigDecimal object on heap
}
```

### 2. Garbage Collection — Generational Model

**Weak generational hypothesis:** Most objects die young. GC exploits this:

```
OBJECT LIFECYCLE:
new Object() → Eden → [Minor GC: survives?] → Survivor (S0↔S1) → [age threshold] → Old Gen

Minor GC (Young Gen):     Fast, frequent, stop-the-world but brief (~1-10ms)
Major GC (Old Gen):       Slower, infrequent, longer pauses
Full GC (everything):     Expensive, last resort — can be seconds on large heaps
```

**How Minor GC works:**
1. New objects allocate in **Eden**
2. When Eden fills → Minor GC triggers
3. Live objects in Eden + current Survivor space → copied to other Survivor space
4. Age counter increments for each survival
5. Objects surviving enough GCs (default threshold ~15) → promoted to **Old Gen**
6. Dead objects = zero cost (no deallocation — Eden is simply wiped)

### 3. GC Algorithms

| GC | When to use | Pause | How it works |
|----|-------------|-------|-------------|
| **Serial** | Small apps, single core | Full STW | Single-thread mark-sweep-compact |
| **Parallel** (Throughput) | Batch processing, max throughput | STW but parallel | Multiple GC threads |
| **G1** (default since J9) | General purpose, balanced | Target pause time | Region-based, concurrent marking, mixed collections |
| **ZGC** (Java 15+) | Ultra-low latency | <10ms regardless of heap size | Concurrent, colored pointers, load barriers |
| **Shenandoah** | Low latency (RedHat) | <10ms | Concurrent compaction with Brooks pointers |

```java
// G1 (default) — good for most applications
// Divides heap into equal-sized regions (~1-32MB each)
// Tracks "garbage-first" — collects regions with most garbage first
// -XX:MaxGCPauseMillis=200 (target pause time)

// ZGC — for latency-sensitive services
// -XX:+UseZGC
// Pauses < 10ms even with terabyte heaps
// Trades throughput for latency

// Key tuning flags:
// -Xms4g -Xmx4g           → set heap (equal = avoid resize pauses)
// -XX:+UseG1GC             → select G1
// -XX:MaxGCPauseMillis=200 → G1 pause target
// -XX:+UseZGC              → select ZGC
// -XX:NewRatio=2           → Old:Young = 2:1
// -Xlog:gc*                → GC logging
```

### 4. Class Loading

```
                    Bootstrap ClassLoader
                    (core JDK: java.lang, java.util)
                           │ parent delegation
                    Platform ClassLoader
                    (java.sql, javax.*)
                           │ parent delegation
                    Application ClassLoader
                    (your classpath)
                           │ parent delegation
                    Custom ClassLoaders
                    (app servers, OSGi, hot-deploy)
```

**Parent-first delegation:** When a class is requested:
1. Ask parent loader first (all the way up to Bootstrap)
2. If parent can't find it → current loader tries
3. If nobody finds it → `ClassNotFoundException`

**This ensures:** `java.lang.String` is always loaded by Bootstrap — you can't replace core classes.

**Lifecycle:** Load (read .class bytes) → Link (verify bytecode, prepare static fields, resolve references) → Initialize (run `static {}` blocks, initialize static fields)

### 5. JIT Compilation

```
Source (.java) → javac → Bytecode (.class) → JVM interprets → JIT compiles hot code → Native

// JVM starts by INTERPRETING bytecode (slow but fast startup)
// Method invocation counter tracks "hot" methods
// When threshold reached → JIT compiles to native machine code

// Tiered compilation (default):
// Level 0: Interpreter
// Level 1-3: C1 compiler (fast compile, basic optimisations)
// Level 4: C2 compiler (slow compile, aggressive optimisations: inlining, loop unrolling, etc.)

// JIT optimisations:
// - Method inlining (eliminates call overhead for small/hot methods)
// - Escape analysis (allocate on stack instead of heap if object doesn't escape method)
// - Dead code elimination
// - Loop unrolling
// - Branch prediction hints
```

### 6. OutOfMemoryError Types

| Error Message | Cause | Fix |
|--------------|-------|-----|
| `Java heap space` | Object leak (e.g., unbounded cache, event listeners not removed) | Heap dump → find leak, increase -Xmx |
| `Metaspace` | Class loader leak (redeploys in app servers) | Fix class loader leak, set -XX:MaxMetaspaceSize |
| `Unable to create native thread` | Too many threads (each ~1MB stack) | Reduce thread count, increase OS limits, use thread pools |
| `Direct buffer memory` | NIO ByteBuffer leak | Close buffers, set -XX:MaxDirectMemorySize |
| `GC overhead limit exceeded` | 98%+ time in GC, <2% heap recovered | Memory leak — heap dump analysis |

```java
// Diagnosing OOM:
// 1. Enable heap dump on OOM:
//    -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof
// 2. Analyse with: Eclipse MAT, VisualVM, or jmap
// 3. Look for: largest retained objects, object reference chains to GC root

// Production monitoring:
// JMX → memory pool usage, GC counts, GC pause times
// JFR (Java Flight Recorder) → low-overhead continuous profiling
//    -XX:StartFlightRecording=duration=60s,filename=recording.jfr
```

### 7. Key JVM Flags for Production

```bash
# Memory
-Xms4g -Xmx4g              # Heap: equal min/max avoids resize pauses
-XX:MaxMetaspaceSize=512m   # Cap metaspace (prevent class loader leaks)
-Xss512k                    # Thread stack size

# GC
-XX:+UseG1GC                # G1 (default but explicit is good)
-XX:MaxGCPauseMillis=200    # G1 pause target
-Xlog:gc*:file=gc.log       # GC logging

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/logs/heap.hprof
-XX:+ExitOnOutOfMemoryError  # Crash fast (containers restart automatically)

# Container-aware (default since Java 10)
-XX:+UseContainerSupport     # Respect cgroup memory/cpu limits
-XX:MaxRAMPercentage=75.0    # Use 75% of container memory for heap
```

---

## Rapid-Fire Q&A

### Q1: Heap vs Stack?
**A:** Stack: per-thread, stores method frames, local primitives, references. LIFO, fast, fixed-size (`-Xss`). Heap: shared across all threads, stores objects. Managed by GC. `-Xms` (initial) / `-Xmx` (max).

### Q2: Generational GC — why?
**A:** Weak generational hypothesis: most objects die young. Young Gen (Eden + Survivor spaces): collected frequently, fast (Minor GC). Old Gen: long-lived objects, collected less often, expensive (Major/Full GC). This makes GC efficient — most work targets a small area.

### Q3: Modern GC algorithms?
**A:** G1 (default since Java 9) — region-based, targets pause time. ZGC — concurrent, <10ms pauses regardless of heap size. Shenandoah — similar to ZGC. Parallel GC — throughput-optimised for batch jobs.

### Q4: What's a stop-the-world pause?
**A:** All application threads are paused while GC runs. Minor GC: ~1-10ms. Full GC: can be seconds on large heaps. ZGC/Shenandoah minimise this with concurrent collection.

### Q5: What's class loading?
**A:** Load → Link (verify, prepare, resolve) → Initialise. Three class loaders: Bootstrap (core JDK), Platform (ext), Application (classpath). Parent-first delegation model.

### Q6: What's JIT compilation?
**A:** JVM starts interpreting bytecode, then JIT-compiles hot methods to native machine code. C1 (client, fast compile) → C2 (server, optimised). Tiered compilation uses both. Hot code runs at near-native speed.

### Q7: What's `OutOfMemoryError` — types?
**A:** `Java heap space` — object leak. `Metaspace` — class loader leak. `Unable to create native thread` — too many threads. `Direct buffer memory` — NIO buffer leak. `GC overhead limit exceeded` — 98%+ time in GC, <2% heap recovered.

### Q8: `-Xms` vs `-Xmx` — why set them equal?
**A:** Avoids heap resize pauses at startup and during load ramps. Standard practice in production and containers.

---

## Can you answer these cold?

- [ ] Heap vs stack — what lives where
- [ ] Generational GC — why it works, Eden → Survivor → Old Gen
- [ ] G1 vs ZGC — when each
- [ ] Five types of `OutOfMemoryError`
- [ ] Class loading delegation model

[← Back to Index](./00_INDEX.md)
