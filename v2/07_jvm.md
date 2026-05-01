# 07 — JVM Internals

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

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
