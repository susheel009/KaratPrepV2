# 07 — JVM Internals

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/07_jvm_doubts.md) · [← Prev: 06 Java 8+](./06_java8_plus.md) · [Next: 08 Design Patterns →](./08_design_patterns.md)
>
> **Priority:** 🟢 Medium · **Related topics:** [01 Concurrency (memory model)](./01_concurrency.md) · [02 Collections (heap layout)](./02_collections.md) · [16 Debugging](./16_debugging.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — what does "the JVM runs your code" actually mean?
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — the life of one object
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — what's stack, what's heap
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Heap vs stack](#41-heap-vs-stack)
   - [4.2 Generations](#42-the-generations-eden-survivor-old)
   - [4.3 Minor / Major / Full GC](#43-minor-vs-major-vs-full-gc)
   - [4.4 Class loading basics](#44-class-loading-basics)
   - [4.5 JIT compilation](#45-jit-compilation--why-java-gets-faster-as-it-runs)
   - [4.6 OOM types](#46-outofmemoryerror--five-flavours)
   - [4.7 Production JVM flags](#47-production-jvm-flags)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 GC algorithm comparison](#51-gc-algorithms-compared)
   - [5.2 G1 in detail](#52-g1-region-based-collection)
   - [5.3 Class loader delegation](#53-class-loader-delegation-and-leaks)
   - [5.4 Tiered JIT](#54-tiered-jit-c1--c2)
   - [5.5 JIT optimisations](#55-jit-optimisations-inlining-escape-analysis)
   - [5.6 Native memory](#56-native-memory-metaspace-code-cache-direct-buffers)
   - [5.7 Diagnostics tools](#57-diagnostics-jmap-jstack-jfr-mat)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Picture a small architectural firm. Each architect has a **desk** — small, organised, theirs alone — where they keep their current sketches, the pencils they're using right now, the calculator they need this minute. When they finish a project, they sweep the desk clean.

Behind the firm is a **shared warehouse** — huge, filled with finished blueprint scrolls, model buildings, and sample materials. Anyone in the office can fetch from it. Things stay in the warehouse until they're no longer needed.

A **janitor** wanders the warehouse periodically. If something in there hasn't been touched by anyone in ages and nobody can locate it on any sketch, the janitor throws it out.

The firm also employs a **freelance translator** in the corner, helping junior architects who only speak the firm's home language read the imported foreign-language reference manuals. At first the translator translates page by page (slow). After watching which manuals get used most, she starts producing fully-translated English versions of the popular ones (fast).

That's the JVM in four pieces.

- **Desk** = each thread's **stack**. Per-thread, small, fast, automatically cleaned when the method returns.
- **Warehouse** = the shared **heap**. All objects live here, all threads can reach them.
- **Janitor** = **garbage collection**. Walks the heap looking for unreachable objects and reclaims their space.
- **Translator** = the **JIT compiler**. Java starts by interpreting bytecode (slow but immediate), then compiles hot methods to native code (fast).

> Once you have those four pictures, every JVM topic — generations, GC algorithms, class loading, OOM types, JIT tiers — is a refinement of one of them. The rest of this file is the refinements.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

Trace the life of one `Order` object from creation to collection:

| Time | Event | Where it lives |
|:--:|---|---|
| t1 | `Order o = new Order(...)` runs in `processBatch()`. JVM allocates space in **Eden** (a sub-region of the young generation in the heap). | Eden |
| t2 | `o` is referenced by a local variable on `processBatch`'s stack frame. The reference (a 4–8 byte pointer) is on the stack; the object is on the heap. | Eden + reference on stack |
| t3 | Eden fills with new allocations from this thread and others. JVM triggers a **Minor GC** (stop-the-world, ~1–10 ms). It scans for live objects in Eden; ours is still referenced. | Eden → copied to Survivor 0 |
| t4 | More work happens. Another Minor GC. Our `Order` is still referenced. It's copied from Survivor 0 to Survivor 1, age counter = 1. | Survivor 1 |
| t5 | After ~15 cycles of surviving (a tunable threshold), the JVM **promotes** our object to the **Old Generation** — long-lived objects. | Old Gen |
| t6 | `processBatch` finishes. Its stack frame pops, taking the local reference with it. If no other reference remains, our `Order` becomes **unreachable**. | Old Gen, but unreferenced |
| t7 | Eventually a **Major GC** runs over the Old Gen. It can't reach our `Order` from any live root → reclaims its memory. | gone |

What this tells you about the design:

- **Most objects die young** — that's the *weak generational hypothesis*. So GC concentrates effort on Eden, where it's cheapest. Most allocations never see a Survivor space, let alone Old Gen.
- **Stack vs heap is automatic.** Local variables and method frames clean up when the method returns. The heap needs the janitor.
- **Stop-the-world** Minor GCs are usually so brief you don't notice. Full GCs over a giant Old Gen can be seconds. That's what the modern algorithms (G1, ZGC, Shenandoah) are designed to avoid.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
public class JvmDemo {
    static int counter = 0;                       // STATIC field — lives in class metadata (Metaspace)

    public static void main(String[] args) {
        int local = 10;                            // primitive on the STACK (in main's frame)
        String name = "Alice";                     // reference on stack; "Alice" in the String pool (heap)
        Order order = new Order("AAPL", 100);      // reference on stack; Order object on the HEAP

        process(order, local);                      // pushes a new frame onto main's stack
        // when process returns, its frame pops — its locals are gone
    }

    static void process(Order o, int qty) {
        // 'o' and 'qty' are this frame's local variables — on the stack
        // 'o' POINTS to a heap object (the Order). The qty value lives directly on the stack.
        BigDecimal cost = new BigDecimal(qty * 10);
        // 'cost' reference on stack; BigDecimal object on the heap
        counter++;                                  // accesses static field — Metaspace lookup, then heap update
    }
}

class Order { String symbol; int qty;
    Order(String s, int q) { symbol = s; qty = q; }
}
```

What just happened: **values** of primitives (`int local`, `int qty`) sit directly on the stack frame. **References** (`order`, `o`, `name`, `cost`) are pointers on the stack pointing into the heap, where the actual `Order`, `BigDecimal`, and pooled `String` objects live. When `process` returns, its frame is discarded — but the heap objects remain until GC reclaims them.

---

## 4. Build Up — Practical Patterns

### 4.1 Heap vs stack

| Aspect | Stack | Heap |
|---|---|---|
| What's stored | Method frames, local primitives, references | Objects, arrays |
| Per-thread? | ✅ each thread has its own | Shared by all threads |
| Size | Small (`-Xss`, default ~512 KB – 1 MB) | Large (`-Xms` / `-Xmx`) |
| Speed | Very fast (LIFO push/pop) | Slower (GC-managed allocation) |
| Cleanup | Automatic on method return | Garbage collector |
| Overflow | `StackOverflowError` (deep recursion) | `OutOfMemoryError: Java heap space` |

```
JVM Process
├── Heap (-Xms / -Xmx)
│   ├── Young Generation
│   │   ├── Eden          ← new allocations here
│   │   ├── Survivor 0
│   │   └── Survivor 1
│   └── Old Generation    ← promoted long-lived objects
├── Metaspace (native memory) — class metadata, method bytecode
├── Thread stacks (-Xss per thread)
├── Code Cache — JIT-compiled native code
└── Direct buffers — NIO ByteBuffer.allocateDirect
```

### 4.2 The generations: Eden, Survivor, Old

```
NEW Order      Eden       Survivor 0/1     Old Gen
   ⤵          ┌───┐       ┌─────┐         ┌─────────────┐
allocate ───▶ │   │  GC ▶ │aged │  GC ▶  │  long-lived │
              │   │  ▶    │1, 2 │   ...   │             │
              └───┘       └─────┘         └─────────────┘
```

- **Eden** — where every new object is born. Minor GCs scan this constantly.
- **Survivor 0 / 1** — two equal-sized regions. On each Minor GC, live objects move from Eden + the active Survivor into the *other* Survivor (a copying collector — fragmentation-free).
- **Old Generation** — for long-lived objects. Higher promotion threshold (default ~15 surviving Minor GCs).

Why two survivor spaces? Copying collection requires a target region, and you alternate which is "from" and "to" each cycle.

### 4.3 Minor vs Major vs Full GC

| Type | Scope | Frequency | Pause | Algorithm |
|---|---|---|---|---|
| **Minor GC** | Young gen only | Often | Short (1–10 ms) | Stop-the-world copying |
| **Major GC** | Old gen | Rare | Long (100 ms – seconds, depends on heap) | Stop-the-world (Parallel) or concurrent (G1, ZGC) |
| **Full GC** | Whole heap (+ Metaspace) | Very rare — last resort | Longest | Concurrent collectors try hard to avoid |

> 💡 "Full GC" in production is almost always a sign of trouble — either a memory leak, a Metaspace leak, or pathological allocation pressure. Never normal under modern collectors.

### 4.4 Class loading basics

```
                    Bootstrap ClassLoader
                    (core JDK: java.lang, java.util)
                              │ parent-first delegation
                    Platform ClassLoader
                    (java.sql, javax.*)
                              │
                    Application ClassLoader
                    (your classpath)
                              │
                    Custom ClassLoaders
                    (app servers, OSGi, hot reload)
```

**Parent-first delegation** — when a class is requested:
1. Ask parent first (recursively up to Bootstrap).
2. If parent doesn't have it → current loader tries.
3. If nobody → `ClassNotFoundException`.

This guarantees `java.lang.String` is *always* loaded by Bootstrap — no rogue classpath jar can replace core types.

**Lifecycle:** Load (read .class bytes) → Link (verify bytecode, prepare static fields, resolve references) → Initialize (run `static {}` blocks).

### 4.5 JIT compilation — why Java gets faster as it runs

```
.java   ── javac ──▶  .class (bytecode)
                              │
                              ▼
                 ┌──────────────────────┐
                 │ Interpreter           │   slow but instant — used for cold methods
                 │ counts invocations    │
                 └──────────────────────┘
                              │ method becomes "hot" (counter > threshold)
                              ▼
                 ┌──────────────────────┐
                 │ JIT Compiler          │   compiles bytecode → native machine code
                 │ (C1 → C2 tiered)      │   stores in the Code Cache
                 └──────────────────────┘
                              │
                              ▼
                       Native code runs
```

Method-invocation counters track which methods are "hot". Once a counter passes the threshold, the JIT compiles the method to native code and stores it in the Code Cache. Subsequent calls execute the native code directly. Java programs are typically slow for the first few seconds (warm-up) and dramatically faster afterwards.

### 4.6 OutOfMemoryError — five flavours

| Message | Cause | Fix |
|---|---|---|
| `Java heap space` | Object leak (unbounded cache, listener not removed, ThreadLocal not cleaned) | Heap dump → find leak; raise `-Xmx` if genuinely under-sized |
| `Metaspace` | Class loader leak (e.g. redeploys in app servers) | Find leak; cap with `-XX:MaxMetaspaceSize` |
| `unable to create native thread` | Too many threads (~1 MB OS stack each) | Use thread pools; lower `-Xss`; raise OS limits |
| `Direct buffer memory` | NIO `ByteBuffer.allocateDirect` not freed | Close buffers; `-XX:MaxDirectMemorySize` |
| `GC overhead limit exceeded` | 98%+ time in GC, < 2% heap reclaimed | Memory leak — heap dump |

### 4.7 Production JVM flags

```bash
# Memory
-Xms4g -Xmx4g                       # equal min/max — avoid resize pauses under load
-XX:MaxMetaspaceSize=512m           # cap metaspace (catch class loader leaks early)
-Xss512k                            # per-thread stack

# GC
-XX:+UseG1GC                        # G1 (default Java 9+ but explicit is good)
-XX:MaxGCPauseMillis=200            # G1 pause target
-Xlog:gc*:file=gc.log:time,uptime    # GC logging (Java 9+ unified logging)

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heap.hprof
-XX:+ExitOnOutOfMemoryError         # crash fast — let the orchestrator restart

# Container-aware (default since Java 10)
-XX:+UseContainerSupport             # respect cgroup CPU/memory limits
-XX:MaxRAMPercentage=75.0            # use 75% of container memory for heap
```

---

## 5. Going Deep — Interview-Level Material

### 5.1 GC algorithms compared

| GC | Java version | Pause profile | When to use |
|---|---|---|---|
| **Serial** | always | Full STW, single thread | Small heaps (< 100 MB), single-core, cli tools |
| **Parallel** (Throughput) | always | Full STW, multi-thread | Batch jobs where total throughput matters more than latency |
| **G1** | default since Java 9 | Short STW, target pause time | General-purpose, heaps 4 GB – several hundred GB |
| **ZGC** | production-ready Java 15+ | < 10 ms regardless of heap size, mostly concurrent | Latency-sensitive services, very large heaps (multi-TB) |
| **Shenandoah** | RedHat builds, mainline Java 12+ | < 10 ms, mostly concurrent | Same niche as ZGC |

**G1, ZGC, Shenandoah are all concurrent.** They do most of their work alongside the application threads, with brief STW pauses for unavoidable steps (root scanning, remark).

### 5.2 G1 region-based collection

G1 divides the heap into 1–32 MB **regions** (size auto-calculated from heap size). Each region is tagged dynamically as Eden, Survivor, or Old. Some regions are special:

- **Humongous regions** — for objects > 50% of region size, bypass Eden directly.
- **Free regions** — pool of unused regions ready for reassignment.

G1 **predicts** which regions can be collected within `-XX:MaxGCPauseMillis`. It greedily picks the regions with the most garbage ("garbage-first") that fit the time budget. Old regions are mostly collected concurrently (concurrent marking + mixed collections — pauses are short slivers, not whole-heap stops).

### 5.3 Class loader delegation and leaks

**Parent-first** delegation has a subtle implication: classes from app servers (e.g. Tomcat war redeploy) are loaded by a per-application class loader. When the app is undeployed, the class loader *should* become unreachable and GC-collectable, freeing all its classes from Metaspace.

**Class loader leak** — anything reachable from a long-lived root that holds a reference to a class loaded by the app's loader keeps the *entire* loader alive (including all its classes). Common culprits:

- A `ThreadLocal` whose value class was loaded by the app loader, sitting in a thread-pool thread that survives redeploys.
- Static caches in *system* libraries that intern app-loaded classes (`Logger.getLogger(MyClass.class)`).
- JDBC driver registration (`DriverManager` keeps a strong ref).

Symptom: Metaspace usage climbs after each redeploy until OOM. Fix: heap dump → look for `ClassLoader` instances reachable from GC roots after undeploy.

### 5.4 Tiered JIT (C1 → C2)

Hotspot ships two JIT compilers:

- **C1 (client)** — fast compile, basic optimisations. Good for code that's hot but not hot-hot.
- **C2 (server)** — slow compile, aggressive optimisations (inlining, escape analysis, loop unrolling, branch profiling). For genuinely hot code paths.

Tiered compilation (default in modern JVMs):
1. Method runs in interpreter, counters tick.
2. Counter crosses C1 threshold → C1 compiles. C1 keeps profiling.
3. Counter crosses C2 threshold → C2 compiles using C1's profile data.
4. C2's code replaces C1's in the Code Cache.

**Deoptimisation** — if a C2 assumption (e.g. "this virtual call always reaches `Foo.method`") is later violated, the JVM throws away the compiled code and falls back to the interpreter. Subtle effects on performance under heavy class loading or rare branches.

### 5.5 JIT optimisations: inlining, escape analysis

A few wins worth knowing by name:

**Method inlining** — call to a small method gets replaced with the method's body at the call site. Eliminates call overhead and exposes more optimisation opportunities (the inlined code can be specialised based on caller context).

**Escape analysis** — the JIT proves an object never escapes the method that allocates it. Then it can:
- **Stack-allocate** — put the object directly on the stack frame (no GC, no heap).
- **Scalar replace** — break the object into individual variables in registers.
- **Eliminate locks** — if the object is never visible to other threads, lock acquisition is skipped.

**Loop unrolling, dead code elimination, branch prediction hints, vectorisation (Java 16+ Vector API)** round out the toolkit.

### 5.6 Native memory: Metaspace, code cache, direct buffers

These live **outside** the Java heap (`-Xmx` doesn't bound them):

- **Metaspace** — class metadata, method bytecode, constant pool. Replaced PermGen in Java 8. Default unbounded — set `-XX:MaxMetaspaceSize` in production.
- **Code Cache** — JIT-compiled native code. If full, JIT stops and the JVM falls back to interpretation. Watch with `-XX:ReservedCodeCacheSize`.
- **Direct buffers** (`ByteBuffer.allocateDirect`) — off-heap memory for NIO. Bounded by `-XX:MaxDirectMemorySize`.
- **Thread stacks** — `~1 MB × thread count`. Unbounded in practice; capped only by OS limits and `-Xss`.

**Container memory = heap + metaspace + code cache + direct + stacks + native (JNI/JIT internals).** When sizing containers, leave 25–30% headroom over `-Xmx`.

### 5.7 Diagnostics: jmap, jstack, JFR, MAT

| Tool | What it does | When |
|---|---|---|
| `jstack <pid>` | Thread dump — every thread's stack trace | App is hung or "slow"; spot deadlocks, blocked threads |
| `jmap -heap <pid>` | Heap usage summary | Quick heap-size check |
| `jmap -dump:live,format=b,file=heap.hprof <pid>` | Full heap dump | Memory leak investigation |
| **JFR (Java Flight Recorder)** | Continuous low-overhead profiling | Production profiling — `-XX:StartFlightRecording=duration=60s,filename=rec.jfr` |
| **Eclipse MAT / VisualVM** | Heap dump analysis | Open `.hprof`, find leak suspects via dominator tree |
| `jcmd <pid> ...` | One-stop replacement for jmap/jstack/JFR | Modern preferred CLI |

```bash
# Common workflow
jcmd <pid> Thread.print                                  # = jstack
jcmd <pid> GC.heap_dump /tmp/heap.hprof                   # = jmap dump
jcmd <pid> JFR.start duration=60s filename=rec.jfr        # = start JFR
jcmd <pid> VM.native_memory summary                       # native memory tracking (NMT)
```

---

## 6. Memory Aids

### Decision tree: "is my OOM a heap, metaspace, or thread issue?"

```
Read the error message — it names the OOM type.

"Java heap space"
  → heap leak. Take a heap dump (-XX:+HeapDumpOnOutOfMemoryError), open in MAT.
  → If genuinely large workload, raise -Xmx.

"Metaspace"
  → class loader leak (very likely, especially in app servers with redeploys).
  → Check: ThreadLocals, static caches keyed on app classes, JDBC driver registrations.

"unable to create native thread"
  → Too many threads. Use a pool. Reduce -Xss.

"Direct buffer memory"
  → NIO leak. Find what's allocating direct ByteBuffers and not closing.

"GC overhead limit exceeded"
  → essentially "heap leak with extra steps". Treat as heap OOM.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Heap vs stack?" | Per-thread vs shared | "Stack: method frames + locals + refs. Heap: objects + arrays." |
| "Why generational GC?" | Most objects die young | "Eden + survivors collected often (cheap); old gen rarely (expensive)." |
| "G1 vs ZGC?" | Heap size, pause budget | "G1 default, balanced. ZGC for very large heaps with strict pause needs (< 10 ms)." |
| "What's a Full GC?" | Whole heap STW | "Last-resort under modern collectors. Frequent Full GCs = leak or undersized." |
| "Why doesn't my heap dump show the leak?" | Look at retained size, not shallow | "MAT's dominator tree → leak suspects." |
| "Why does Java get faster after warm-up?" | JIT compiles hot methods | "Tiered: interpreter → C1 → C2." |
| "What's escape analysis?" | Stack allocation if object doesn't escape | "Proven non-escape → stack allocate or scalar-replace, no GC." |
| "What's `-Xms` and `-Xmx`?" | Initial / max heap | "Set them equal in production to avoid resize pauses." |

### Three anchor pictures

1. **Stack = your desk; heap = the warehouse.** Janitor (GC) only visits the warehouse.
2. **Eden → Survivor → Old.** Most objects don't reach Survivor, let alone Old.
3. **Java starts slow, gets fast.** JIT translates hot methods to native code.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: Heap vs Stack?
**A:** Stack: per-thread, stores method frames + locals + references. LIFO, fast, fixed-size (`-Xss`). Heap: shared, stores objects + arrays. GC-managed. `-Xms`/`-Xmx` size it.

### Q2: Why generational GC?
**A:** Weak generational hypothesis: most objects die young. Young Gen (Eden + survivors) is collected often, fast (Minor GC). Old Gen is collected rarely, slowly (Major/Full GC). Concentrating effort where most of the death happens makes GC efficient.

### Q3: Modern GC algorithms?
**A:** G1 (default since Java 9) — region-based, target pause time. ZGC — concurrent, < 10 ms pauses regardless of heap. Shenandoah — similar to ZGC. Parallel — throughput-optimised. Serial — small heaps.

### Q4: What's a stop-the-world pause?
**A:** All application threads paused while the GC works. Minor GC: ~1–10 ms. Full GC: 100 ms to seconds depending on heap. Concurrent collectors (G1, ZGC, Shenandoah) shrink STW to brief slivers.

### Q5: What's class loading?
**A:** Three steps: Load (read .class bytes), Link (verify, prepare static fields, resolve references), Initialize (run `static {}` blocks). Bootstrap → Platform → Application → custom. Parent-first delegation.

### Q6: What's JIT compilation?
**A:** JVM starts by interpreting bytecode. Hot methods get compiled to native machine code by C1 (fast compile) and later C2 (heavy optimisation). Tiered compilation uses both. Native code runs from the Code Cache.

### Q7: Five `OutOfMemoryError` messages and their causes?
**A:** `Java heap space` (object leak), `Metaspace` (class-loader leak), `unable to create native thread` (too many threads), `Direct buffer memory` (NIO leak), `GC overhead limit exceeded` (98 %+ time in GC).

### Q8: `-Xms` vs `-Xmx`?
**A:** Initial vs max heap. Set them equal in production to avoid resize pauses during load ramps.

### Q9: What's Metaspace?
**A:** Native (off-heap) memory for class metadata, method bytecode, constant pool. Replaced PermGen in Java 8. Default unbounded — cap with `-XX:MaxMetaspaceSize`.

### Q10: Container-aware JVM?
**A:** Default since Java 10. JVM reads cgroup limits and sizes the heap proportionally (`-XX:MaxRAMPercentage=75.0` is typical). Without this, JVM thinks it has the whole host's RAM and OOMs the container.

### Q11: How do you take a heap dump?
**A:** `-XX:+HeapDumpOnOutOfMemoryError` (automatic on OOM) or `jcmd <pid> GC.heap_dump path.hprof`. Analyse with Eclipse MAT — dominator tree shows which objects retain the most memory.

### Q12: Escape analysis?
**A:** JIT proves an object doesn't escape the method that allocates it. Enables stack allocation, scalar replacement, lock elimination — turning "heap allocation + GC" into "register variable" in the best case.

### Q13: G1's "garbage-first" idea?
**A:** Heap divided into 1–32 MB regions. JVM tracks how much garbage is in each. G1 selects the regions with the most garbage that fit within `-XX:MaxGCPauseMillis`. Concentrates work where it pays best.

### Q14: What's the Code Cache?
**A:** Off-heap region holding JIT-compiled native code. Sized via `-XX:ReservedCodeCacheSize`. If full, JIT stops; the JVM falls back to interpretation — major performance regression.

### Q15: When would you choose ZGC over G1?
**A:** Large heaps (10s of GB to TB) with strict pause requirements. ZGC pauses are < 10 ms regardless of heap size. Trade-off: lower throughput than G1 (a few percent), and uses more CPU for concurrent work.

### Q16: How do you find a memory leak in production?
**A:** (1) Verify symptoms via JMX / GC logs / monitoring. (2) Take a heap dump (`-XX:+HeapDumpOnOutOfMemoryError` or `jcmd`). (3) MAT or VisualVM, dominator tree, look for largest retainers. (4) Trace from suspect to GC root to identify the leak path. (5) Fix and verify.

---

### Key Code/Flag Patterns

**Production starter set**
```bash
-Xms4g -Xmx4g -XX:MaxMetaspaceSize=512m -Xss512k
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/heap.hprof
-XX:+ExitOnOutOfMemoryError
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```

**Container-aware**
```bash
-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0
```

**Diagnostic snapshot**
```bash
jcmd <pid> Thread.print > threads.txt
jcmd <pid> GC.heap_dump /tmp/heap.hprof
jcmd <pid> JFR.start duration=60s filename=/tmp/profile.jfr
```

---

## 8. Self-Test

**Easy**
- [ ] Where do primitive locals live — stack or heap?
- [ ] What's the difference between `-Xms` and `-Xmx`?
- [ ] Is Metaspace inside the heap?
- [ ] What does the JIT do?

**Medium**
- [ ] Trace one object through Eden → Survivor → Old Gen step by step.
- [ ] Why is "Full GC" rare in healthy production systems?
- [ ] What's parent-first class-loader delegation, and why does it matter?
- [ ] List the five `OutOfMemoryError` messages and the typical cause of each.

**Hard**
- [ ] Compare G1, ZGC, and Shenandoah — when would you pick each?
- [ ] Describe a class-loader leak and one common cause.
- [ ] What's escape analysis? Give an example of an object the JIT could stack-allocate.
- [ ] What's the difference between the C1 and C2 compilers?
- [ ] Diagnose: Metaspace usage rises after every Tomcat redeploy. Where do you start?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Heap** | Shared object memory, GC-managed. |
| **Stack** | Per-thread memory for method frames, locals, references. |
| **Eden** | Sub-region of the young gen where new objects are allocated. |
| **Survivor** | Two equal sub-regions where Minor-GC survivors are copied. |
| **Old Generation** | Heap region for long-lived objects (promoted from Survivor). |
| **Metaspace** | Native memory for class metadata. Not part of the heap. |
| **Minor GC** | Young-gen-only collection. Frequent, brief STW. |
| **Major GC** | Old-gen collection. Infrequent, longer pause. |
| **Full GC** | Whole-heap collection. Last-resort under modern collectors. |
| **STW (stop-the-world)** | Pause where all app threads are stopped. |
| **JIT (Just-In-Time)** | Runtime bytecode-to-native compiler. |
| **C1 / C2** | Tiered JIT compilers — fast-compile / heavy-optimisation. |
| **Code Cache** | Off-heap region for JIT-compiled native code. |
| **Class loader** | Component that loads .class bytes and defines the Class. |
| **Parent-first delegation** | Class-loader rule: ask parent before trying yourself. |
| **Class-loader leak** | Strong reference holds a per-app class loader alive after redeploy. |
| **Escape analysis** | JIT proof that an object doesn't leak the method — enables stack allocation. |
| **Direct buffer** | Off-heap ByteBuffer used by NIO. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/07_jvm_doubts.md) · [← Prev: 06 Java 8+](./06_java8_plus.md) · [Next: 08 Design Patterns →](./08_design_patterns.md)

[↑ Back to top](#07--jvm-internals)
