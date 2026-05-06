# 01 — Concurrency & Threading

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/01_concurrency_doubts.md) · [Next: 02 Collections →](./02_collections.md)
>
> **Priority:** 🔴 Critical · **Related topics:** [02 Collections (concurrent maps)](./02_collections.md) · [07 JVM (memory model)](./07_jvm.md) · [16 Debugging (race conditions)](./16_debugging.md) · [17 Debugging Problems](./17_debugging_problems.md)
>
> **Practice:** [Karat debugging problems](./Practice/karat-debugging-problems.md) · [Karat solutions](./Practice/karat-solutions.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — race conditions in plain English
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — the lost-update example, narrated
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — minimal counter race + fix
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Thread lifecycle](#41-the-thread-lifecycle-your-q2-answered)
   - [4.2 Creating threads](#42-creating-and-starting-a-thread-your-q3-q4-answered)
   - [4.3 join, main, daemon](#43-join-the-main-thread-and-daemon-threads-your-q6-q7-answered)
   - [4.4 synchronized — what it locks](#44-synchronized--modifier-or-block-and-what-it-actually-locks-your-q5-q9-answered)
   - [4.5 Reentrant locks](#45-reentrant-why-a-thread-can-re-enter-its-own-lock-your-q8-answered)
   - [4.6 volatile](#46-volatile--visibility-not-atomicity-your-q10-answered)
   - [4.7 Atomic classes](#47-atomic-classes--when-synchronized-is-overkill)
   - [4.8 ExecutorService](#48-executorservice--never-make-raw-threads-in-production)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 JMM and happens-before](#51-the-java-memory-model-and-happens-before)
   - [5.2 CAS](#52-cas-compare-and-swap--the-engine-under-atomic)
   - [5.3 ReentrantLock and friends](#53-reentrantlock-and-friends)
   - [5.4 wait/notify and Condition](#54-wait--notify-and-condition)
   - [5.5 CompletableFuture](#55-completablefuture--async-composition)
   - [5.6 Synchronisers](#56-synchronisers-latch-barrier-semaphore)
   - [5.7 BlockingQueue](#57-blockingqueue-and-the-producer-consumer-pattern)
   - [5.8 ThreadLocal](#58-threadlocal--per-thread-storage-mind-the-leak)
   - [5.9 ForkJoinPool](#59-forkjoinpool-and-work-stealing)
   - [5.10 Virtual threads](#510-virtual-threads-java-21)
   - [5.11 Deadlock](#511-deadlock--the-four-conditions-and-how-to-break-them)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **How to read this file.** §1–§3 build the picture from scratch. §4 is everyday tools. §5 is interview depth. §6–§9 are recall, drill, and lookup.
> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Picture a busy restaurant kitchen.

There's **one chef** at first. She does everything in order: chop, then sauté, then plate. One thing at a time. Slow but predictable — every order goes out correctly, just not very fast.

To go faster, the owner hires more chefs. Now five chefs share **one kitchen** — same fridge, same knives, same notepad on the wall where they tally "orders sent out tonight". Total throughput goes way up. But three new headaches show up:

1. **Two chefs reach for the same knife.** Whoever grabs it first uses it; the other waits. If they both grab at the *exact* same instant, they fight over it. In code, the "knife" is a piece of shared data — a counter, a list, a customer record.
2. **The notepad gets miscounted.** Chef A glances at the notepad, sees "5 orders sent". Before she writes "6", Chef B also glances, also sees "5", and also writes "6". Two orders went out, but the tally only moved by 1. **The count is wrong, and nobody noticed.**
3. **Two chefs deadlock at the pantry.** Chef A is holding the salt and waiting for the pepper. Chef B is holding the pepper and waiting for the salt. Neither will let go until the other does. The whole kitchen freezes.

That's concurrency in one paragraph. **Threads** are the chefs. **Shared variables** are the utensils and notepad. The three headaches above are the three problems we'll spend the rest of the file solving:

- **Race condition** — two threads stomp on shared data → wrong result.
- **Visibility** — one thread updates a value, but other threads keep reading a stale copy from their own private cache.
- **Deadlock** — two threads each hold what the other needs → freeze.

> Once you've internalised those three problems, every Java concurrency tool (`synchronized`, `volatile`, `Atomic*`, `ReentrantLock`, `ExecutorService`, `CompletableFuture`, virtual threads…) is just a different way to fix one of them more efficiently.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

Let's actually watch a race condition happen, in slow motion, with no Java syntax.

**Scenario:** The notepad on the wall says **5**. Two chefs each just sent out one more order, so the count *should* end up at **7**.

What "increment the notepad" actually means in three steps:
1. **Read** the current number off the notepad (your eyes).
2. **Add 1** to it (in your head).
3. **Write** the new number back on the notepad (your hand).

Now interleave the two chefs:

| Time | Chef A | Chef B | Notepad |
|:----:|--------|--------|:-------:|
| t1 | reads 5 | | 5 |
| t2 | | reads 5 ← **before A writes!** | 5 |
| t3 | thinks "5 + 1 = 6" | | 5 |
| t4 | | thinks "5 + 1 = 6" | 5 |
| t5 | writes 6 | | 6 |
| t6 | | writes 6 ← **overwrites A** | 6 |

Final notepad: **6**. Two orders went out. **One increment was lost** and there's no exception, no error log, nothing — just a quietly wrong number. This is a race condition.

**The fix in plain English:** put a single bathroom-door lock on the notepad. Before you read it, you grab the door handle (lock). Anyone else who wants the notepad has to wait at the door. When you've finished writing, you let go (unlock). Now the read–add–write happens as one uninterruptible chunk. Total at the end: 7. Correct.

That door handle is exactly what `synchronized` is in Java. Now the code.

---

## 3. First Code (minimal, every non-obvious line commented)

Here is the kitchen scene as Java. Two threads each increment a shared counter 1000 times. Expected total: 2000.

```java
public class CounterDemo {
    // The "notepad on the wall" — shared between threads.
    static int counter = 0;

    public static void main(String[] args) throws InterruptedException {
        // Define the work: increment counter 1000 times.
        Runnable work = () -> {
            for (int i = 0; i < 1000; i++) {
                counter++;          // read → add → write — NOT atomic
            }
        };

        // Two chefs. Each will run `work`.
        Thread t1 = new Thread(work);
        Thread t2 = new Thread(work);

        t1.start();                 // .start() spawns a NEW OS thread.
        t2.start();                 //   (.run() would just run on THIS thread — wrong.)

        t1.join();                  // Main thread waits here until t1 finishes.
        t2.join();                  // Then waits until t2 finishes.

        System.out.println(counter); // Expected 2000. You'll see 1300, 1842, 1999, ... random.
    }
}
```

If you run this, you'll almost never get 2000. You'll get something *close* to 2000 but not equal — that's the lost-update pattern from §2, just at a much smaller time scale (nanoseconds instead of seconds).

**Now the fix.** Wrap the unsafe step inside a `synchronized` block. The block uses an object — any object — as the "door handle":

```java
public class CounterDemo {
    static int counter = 0;
    static final Object lock = new Object();   // the "door handle"

    public static void main(String[] args) throws InterruptedException {
        Runnable work = () -> {
            for (int i = 0; i < 1000; i++) {
                synchronized (lock) {           // grab the handle...
                    counter++;                  // ...do the read-add-write...
                }                               // ...let go (auto on block exit).
            }
        };

        Thread t1 = new Thread(work);
        Thread t2 = new Thread(work);
        t1.start(); t2.start();
        t1.join();  t2.join();
        System.out.println(counter);            // Always 2000 now.
    }
}
```

**What changed:** every time a thread reaches `synchronized (lock) { ... }`, it must own the `lock` object before entering. Only one thread can own it at a time. Other threads pile up at the door (state: `BLOCKED`). When the owning thread exits the block — even if it throws — the lock is released and the next thread proceeds.

That's the entire mental model. Everything below is variations: different doors, different handles, different ways to wait.

---

## 4. Build Up — Practical Patterns

### 4.1 The thread lifecycle (your **Q2** answered)

A Java thread is always in exactly one of these states (`Thread.State` enum):

```
NEW ──start()──▶ RUNNABLE ◀──┬── BLOCKED       (waiting to grab a synchronized lock)
                              ├── WAITING       (wait(), join(), park() with no timeout)
                              ├── TIMED_WAITING (sleep(ms), wait(ms), join(ms))
                              │
                              ▼
                          TERMINATED            (run() returned, or threw uncaught)
```

| State | Got here by… | Leaves when… |
|-------|--------------|--------------|
| `NEW` | `new Thread(...)` — not started yet | you call `.start()` |
| `RUNNABLE` | `.start()` was called | the thread blocks, waits, or finishes |
| `BLOCKED` | tried to enter `synchronized` but lock is held | the lock becomes free |
| `WAITING` | called `wait()`, `join()`, `LockSupport.park()` | someone calls `notify()` / target thread ends / `unpark()` |
| `TIMED_WAITING` | `sleep(ms)`, `wait(ms)`, `join(ms)` | the timeout expires *or* you get notified |
| `TERMINATED` | `run()` returned or threw | — (a thread cannot be restarted) |

> **Two important nuances:**
> - There is **no `RUNNING` state in Java's model.** `RUNNABLE` covers both "scheduled and running on a CPU right now" and "ready to run but the OS hasn't picked me yet". Java intentionally hides that distinction because it's an OS-scheduler decision the JVM doesn't fully control.
> - **A thread does *not* have to terminate.** A web server worker thread might cycle `RUNNABLE ↔ WAITING ↔ RUNNABLE` forever, waiting for the next request. "Cycle forever" is a perfectly normal, healthy thread lifecycle.

### 4.2 Creating and starting a thread (your **Q3, Q4** answered)

There are three ways to create a thread. **Implementing `Runnable` is the default; extending `Thread` is rare and almost always wrong.**

```java
// Way 1: extend Thread.  Avoid — it burns your one shot at extending a class
//        on the threading machinery, with no real benefit.
class MyTask extends Thread {
    @Override public void run() { System.out.println("hello from " + getName()); }
}
new MyTask().start();

// Way 2: implement Runnable (or pass a lambda — Runnable is a functional interface).
//        This is the default.  Your task class stays free to extend something useful.
Thread t = new Thread(() -> System.out.println("hello"));
t.start();

// Way 3: Callable<T> + Future<T>.  Use this when the task RETURNS a value
//        or needs to throw a checked exception.
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<String> f = pool.submit(() -> {           // Callable<String>
    Thread.sleep(200);
    return "result";
});
System.out.println(f.get());   // .get() blocks until the task is done
pool.shutdown();
```

**Multiple threads, predictably:**

```java
List<Thread> chefs = new ArrayList<>();
for (int i = 0; i < 5; i++) {
    int id = i;
    Thread chef = new Thread(() -> {
        for (int j = 0; j < 100; j++) {
            System.out.println("chef " + id + " did task " + j);
        }
    });
    chefs.add(chef);
    chef.start();              // 5 chefs, each printing 100 lines = 500 lines total,
}                              // but the lines INTERLEAVE in random order each run.

for (Thread c : chefs) c.join();   // main thread waits until ALL chefs are done
System.out.println("all chefs finished");
```

> ⚠ **Always call `.start()`, never `.run()`.** `.run()` is just a normal method call — it executes `run()` on whichever thread called it. No new thread is created. This is the #1 beginner gotcha.

### 4.3 `.join()`, the main thread, and daemon threads (your **Q6, Q7** answered)

**The main thread:** When the JVM starts your program, it creates one thread automatically and runs your `public static void main(...)` method on it. That's the main thread. Every other thread you create *branches off* from it (or from a thread that branched off from it, etc.).

All threads in one JVM **share the same heap** — the same objects, the same fields. This is the whole reason concurrency is hard. (Compare: separate OS *processes* each have their own memory and can't see each other's variables — much safer, but heavier.)

**`.join()`:** "Pause my thread until that other thread finishes." Without `join()`, the main thread happily walks past the line that started the children and might exit the program before they've done any work.

```java
Thread worker = new Thread(() -> heavyWork());
worker.start();
// main thread keeps running here — worker is doing its thing in parallel
worker.join();          // main now blocks here until worker's run() returns
System.out.println("worker done, continuing main");
```

**Daemon threads:** A "background" thread that the JVM is *willing to abandon*. Normally the JVM stays alive until every non-daemon thread finishes. If the only threads still running are daemons, the JVM exits immediately, killing them mid-task.

```java
Thread housekeeper = new Thread(() -> { while (true) cleanCaches(); });
housekeeper.setDaemon(true);   // MUST be set before .start()
housekeeper.start();
// main returns → JVM exits → housekeeper is killed instantly. No cleanup, no finally.
```

Use daemons for things that should die when the app dies — metrics flushers, log rotators, background heartbeats. Never use them for work that must complete (e.g. flushing a write buffer to disk).

### 4.4 `synchronized` — modifier or block, and what it actually locks (your **Q5, Q9** answered)

`synchronized` is **a modifier on methods** (like `static`, `public`) **and also a block keyword**. So it's a bit of both.

The crucial point: **`synchronized` locks an OBJECT, not a method.** Every Java object has, hidden inside it, a single built-in lock called the *intrinsic monitor*. `synchronized` acquires that lock.

| Form | What gets locked |
|------|------------------|
| `synchronized` instance method | `this` — the specific object the method is called on |
| `synchronized` static method | the `Class` object (e.g. `Account.class`) — shared across **all** instances |
| `synchronized (someObj) { ... }` block | `someObj`'s monitor — finest control |

```java
class Account {
    int balance;

    // Locks `this`. Two threads calling acc1.transfer(...) and acc1.deposit(...)
    // are serialised. But acc1.transfer(...) and acc2.transfer(...) run in PARALLEL —
    // they're locking different `this` objects.
    public synchronized void transfer(int amount) { balance -= amount; }
    public synchronized void deposit(int amount)  { balance += amount; }

    // Locks Account.class. ALL instances share this lock — every thread calling
    // any static synchronized method on Account waits for every other.
    public static synchronized Config getConfig() { return globalConfig; }

    // Block form — lock on a private dedicated object. Best practice for libraries:
    //   - Outsiders can't accidentally lock on `this` and starve your code.
    //   - You can keep critical sections small (only the read-modify-write part).
    private final Object lock = new Object();
    public void update() {
        // ... non-critical work that doesn't need the lock ...
        synchronized (lock) {
            // ... only the actual race-prone bit ...
        }
        // ... more non-critical work ...
    }
}
```

> **Why the OBJECT, not the method?** Because the safety we care about is "two threads aren't reading/writing the same data at the same time". The data lives in the object, not in the method. So Java attaches the lock to the object whose data needs protecting. Two different `Account` instances are two different bank accounts — their state can be modified concurrently, perfectly safely.

### 4.5 Reentrant: why a thread can re-enter its own lock (your **Q8** answered)

A lock is **reentrant** if the thread that already owns it can acquire it *again* without blocking. Java's `synchronized` lock is reentrant.

**Why this matters:** without reentrance, a thread would deadlock itself the moment it called another synchronized method on the same object.

```java
class Account {
    public synchronized void a() {
        b();                       // calls b() — also synchronized on `this`
    }
    public synchronized void b() {
        // OK — this thread already holds the lock from a(), so it just re-enters.
        // Lock count: 2. When b() returns: count drops to 1. When a() returns: 0 → released.
    }
}
```

The JVM keeps a **lock count** on every monitor: increment when the same thread re-enters, decrement on exit, release when it hits 0. Different thread? Still has to wait at the door.

> If reentrance didn't exist, `a()` would call `b()`, `b()` would try to grab the lock, find that "someone" (itself) already holds it, and wait forever. So reentrance isn't a fancy feature — it's the bare minimum to make `synchronized` usable across method calls.

The explicit lock class `ReentrantLock` (covered in §5.3) has the word right in the name — same property.

### 4.6 `volatile` — visibility, NOT atomicity (your **Q10** answered)

To understand `volatile`, you need to know about CPU caches.

When a thread runs on a CPU core, that core has its own tiny private cache. Reads and writes go to the cache; the CPU only flushes the cache to main memory when it feels like it. So:

> Thread A on core 1 sets `running = false`. Thread B on core 2 keeps reading `running` and sees `true`, **forever**, because thread B is reading from core 2's cache, which never got the update from core 1.

This isn't a bug — it's a deliberate hardware optimisation called the *memory hierarchy*. Without it, multi-core CPUs would be massively slower. But it does mean: **a write by one thread is not automatically visible to another thread.**

`volatile` fixes that for one variable:

```java
class Worker {
    volatile boolean running = true;   // <-- volatile

    public void loop() {
        while (running) {              // every read goes to main memory, not the cache
            doOneIteration();
        }
        System.out.println("clean exit");
    }

    public void stop() {
        running = false;               // every write goes to main memory immediately
    }
}
```

**What `volatile` guarantees:**
1. **Visibility** — every read sees the most recent write by *any* thread.
2. **Ordering** — establishes a *happens-before* edge (more on this in §5.1). The compiler/CPU isn't allowed to reorder reads and writes across a volatile access in ways that would break that edge.
3. **Atomic 64-bit reads/writes** — an obscure but real guarantee. Without `volatile`, a `long` or `double` write *may* be split into two 32-bit writes that another thread could observe half-done. With `volatile`, it's atomic.

**What `volatile` does NOT guarantee:**

```java
volatile int count = 0;
count++;                  // STILL BROKEN. Three sub-operations:
                          //   1. READ count  (volatile — sees latest)
                          //   2. ADD 1       (local CPU register)
                          //   3. WRITE count (volatile — publishes)
                          // Another thread can squeeze its own read+write
                          // between 1 and 3 → lost update, exactly like §2.
```

**When is `volatile` enough?** Only when the variable is **written from one thread and just read from others** (typical: a flag), OR when each operation is a single read or single write with no compute step in between.

| You want… | Use |
|-----------|-----|
| Boolean flag, written by one thread, read by many | `volatile boolean` |
| Counter incremented by many threads | `AtomicInteger` (or `synchronized`, or `LongAdder` under high contention) |
| "Set this reference, others read latest" | `volatile T` or `AtomicReference<T>` |
| Compound condition (`if not present, put`) | `synchronized` block, or `ConcurrentHashMap.computeIfAbsent` |

### 4.7 Atomic classes — when `synchronized` is overkill

If all you need is a thread-safe counter, `synchronized` works but is heavier than necessary. Java provides lock-free classes built on a CPU instruction called **CAS** (compare-and-swap, covered in §5.2).

```java
import java.util.concurrent.atomic.*;

AtomicInteger orders = new AtomicInteger(0);
orders.incrementAndGet();          // atomic ++orders, returns new value
orders.addAndGet(5);               // atomic orders += 5, returns new value
orders.compareAndSet(10, 20);      // if orders == 10, set to 20; returns success boolean

AtomicReference<Config> cfg = new AtomicReference<>(initial);
cfg.set(newConfig);                // atomic publish — all readers see new value
Config c = cfg.get();              // atomic read

// Under HEAVY contention (e.g. metric counters hit by hundreds of threads),
// AtomicLong's retry loop becomes a hotspot. LongAdder shards the counter
// across many cells, so threads update different cells and rarely collide.
LongAdder hits = new LongAdder();
hits.increment();                  // O(1) amortised, almost no contention
long total = hits.sum();           // sums all cells — eventually consistent
```

| Tool | When |
|------|------|
| `AtomicInteger` / `AtomicLong` | Counter or simple atomic state, low–medium contention |
| `LongAdder` / `LongAccumulator` | High-contention counter (metrics, telemetry) |
| `AtomicReference<T>` | Atomic swap of an immutable object |
| `AtomicIntegerFieldUpdater` | You want atomic semantics on an existing `int` field without the overhead of an `AtomicInteger` per instance |

### 4.8 ExecutorService — never make raw threads in production

Creating `new Thread(...)` per task is expensive (each platform thread is ~1 MB of stack, plus kernel bookkeeping) and gives you no control over how many run at once. Use a pool.

```java
import java.util.concurrent.*;

// Fixed pool — exactly N worker threads, bounded queue is your job.
ExecutorService pool = Executors.newFixedThreadPool(8);

pool.submit(() -> doWork());                                // Runnable
Future<String> f = pool.submit(() -> computeResult());      // Callable
String r = f.get();                                          // blocks until done

pool.shutdown();                    // stop accepting new tasks; let queued ones finish
// pool.shutdownNow();              // interrupt running tasks, return queued-but-unstarted

// In production: build the executor explicitly so you control the queue and rejection.
ThreadPoolExecutor prod = new ThreadPoolExecutor(
    4,                                              // core threads (always alive)
    16,                                             // max threads (created on burst)
    60, TimeUnit.SECONDS,                           // idle keep-alive for non-core threads
    new ArrayBlockingQueue<>(1000),                 // BOUNDED queue — backpressure
    new ThreadPoolExecutor.CallerRunsPolicy()       // when full: caller runs the task
);
```

**Pool sizing rule of thumb:**

| Workload | Threads | Why |
|----------|--------:|-----|
| CPU-bound (math, parsing) | `cores + 1` | One spare for occasional page faults; more = context-switch waste |
| I/O-bound (HTTP, DB) | `cores × (1 + waitTime / computeTime)` | Threads spend most time blocked, so you need many to keep CPUs busy |
| Mixed | **separate pools** for each | Stops slow I/O tasks from starving CPU work |

> ⚠ **Avoid `Executors.newCachedThreadPool()` in production.** It creates an unbounded number of threads on burst load → out-of-memory. Use it only for short-lived demos / tests.

---

## 5. Going Deep — Interview-Level Material

(You've now got the foundation. From here on, the material is denser because each concept builds on §1–§4.)

### 5.1 The Java Memory Model and "happens-before"

The Java Memory Model (JMM) is the spec that says **when one thread's writes become visible to another thread**. Without it, the JIT compiler and the CPU are free to reorder, cache, and elide reads and writes — your code is correct only by accident.

The JMM defines a *partial order* called **happens-before**. If action A happens-before action B, then B is guaranteed to see A's effects. The rules:

| Rule | Reads as |
|------|----------|
| **Program order** | Within a single thread, statement N happens-before statement N+1. |
| **Monitor lock** | Unlocking a monitor happens-before any later lock of the same monitor (by any thread). |
| **Volatile** | A write to a volatile field happens-before every subsequent read of that field. |
| **Thread start** | `thread.start()` happens-before any action in the started thread. |
| **Thread join** | All actions in a thread happen-before any other thread's successful return from `thread.join()`. |
| **Transitivity** | If A hb B and B hb C, then A hb C. |

**Why this matters in code you'll actually write:** classic *double-checked locking* without `volatile` is broken because of this:

```java
class Config {
    private static Config instance;          // NOT volatile — BROKEN
    public static Config get() {
        if (instance == null) {              // (1)
            synchronized (Config.class) {
                if (instance == null) {
                    instance = new Config(); // (2) — see below
                }
            }
        }
        return instance;                     // (3)
    }
}
```

`new Config()` looks atomic but in bytecode is roughly: (a) allocate memory, (b) write fields, (c) publish reference. The JIT is allowed to reorder (b) and (c), so another thread could pass check (1), see a non-null `instance`, return it, and dereference a Config whose fields are still default zeros.

**Fix:** mark `instance` `volatile`. The volatile write of the reference establishes a happens-before edge that prohibits the reordering.

### 5.2 CAS (compare-and-swap) — the engine under `Atomic*`

CAS is a CPU instruction (`CMPXCHG` on x86, `LDREX/STREX` on ARM) that does atomically:

> "If the current value at this address equals `expected`, replace it with `new` and return success. Otherwise, return failure and leave the value alone."

`AtomicInteger.incrementAndGet()` is implemented as a CAS loop:

```java
// Conceptually:
int cur, next;
do {
    cur  = get();                          // read current value
    next = cur + 1;                        // compute new value
} while (!compareAndSet(cur, next));       // CAS — retry if anyone else changed it
return next;
```

This is **lock-free**: no thread is ever `BLOCKED`. Under low contention, blazing fast (no kernel involvement). Under *very* high contention, threads keep CAS-failing and retrying — wasted CPU; switch to `LongAdder` (§4.7) which avoids the contention entirely by sharding.

### 5.3 `ReentrantLock` and friends

`synchronized` is great for 80% of cases. The other 20% need `ReentrantLock` (or its cousins) because they offer features `synchronized` doesn't:

| Feature | `synchronized` | `ReentrantLock` |
|---------|:--------------:|:---------------:|
| Reentrant | ✅ | ✅ |
| Auto-release on exception | ✅ | ❌ — **you must `unlock()` in `finally`** |
| Try with timeout | ❌ | ✅ `tryLock(t, unit)` |
| Interruptible while waiting | ❌ | ✅ `lockInterruptibly()` |
| Fairness option (FIFO) | ❌ | ✅ `new ReentrantLock(true)` |
| Multiple condition variables on one lock | ❌ (just `wait/notify`) | ✅ `lock.newCondition()` |

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // critical section
} finally {
    lock.unlock();          // YOU MUST do this — synchronized auto-unlocks, this doesn't
}

// tryLock with timeout — useful for breaking deadlocks
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try { /* work */ } finally { lock.unlock(); }
} else {
    // didn't get the lock in 500ms — bail out, retry, log, whatever
}
```

**`ReadWriteLock`** — many readers OR one writer. Use when reads vastly outnumber writes (config, lookup tables):

```java
ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();   try { /* read */ }  finally { rw.readLock().unlock(); }
rw.writeLock().lock();  try { /* write */ } finally { rw.writeLock().unlock(); }
// Multiple threads can hold the read lock simultaneously.
// The write lock is exclusive — blocks readers AND other writers.
```

**`StampedLock`** (Java 8+) — adds *optimistic reads*: read without locking and validate after. Even faster than `ReadWriteLock` for read-heavy workloads. Not reentrant.

### 5.4 `wait` / `notify` and `Condition`

The classic low-level coordination primitive. **Always inside `synchronized`**, on the same monitor:

```java
synchronized (lock) {
    while (!conditionMet) {        // WHILE not IF — see "spurious wakeups" below
        lock.wait();               // releases lock, parks thread; reacquires on wakeup
    }
    // proceed — conditionMet is true and we hold the lock
}

// elsewhere:
synchronized (lock) {
    conditionMet = true;
    lock.notifyAll();              // wake all threads waiting on `lock`
    // notify() wakes one — but you usually want notifyAll() for safety
}
```

> **Why `while`, not `if`?** The JVM is allowed to wake a thread *spuriously* — without anyone calling `notify()`. If you used `if`, you'd proceed even though the condition isn't actually met. `while` re-checks after the wakeup.

`Condition` (from `ReentrantLock`) is the modern replacement: same semantics, but you get **multiple condition variables on one lock** — perfect for a bounded queue with both `notFull` and `notEmpty` conditions.

### 5.5 `CompletableFuture` — async composition

`Future` (Java 5) has only `get()` — you block. `CompletableFuture` (Java 8+) is composable: chain transformations, handle errors, combine results, all without blocking.

```java
CompletableFuture
    .supplyAsync(() -> fetchUser(id), executor)        // start work on `executor`
    .thenApply(user -> user.getEmail())                // sync transform (like map)
    .thenComposeAsync(this::lookupPrefsAsync)          // async transform (like flatMap)
    .thenCombine(fetchPermsAsync(id), this::merge)    // combine with another CF
    .thenAccept(result -> log.info("done: {}", result))// terminal — consumes
    .exceptionally(ex -> { log.error("failed", ex); return null; });

// Wait for many:
CompletableFuture.allOf(cf1, cf2, cf3).join();   // blocks until ALL complete
CompletableFuture.anyOf(cf1, cf2, cf3).join();   // blocks until FIRST completes
```

| Method | Like | Returns |
|--------|------|---------|
| `thenApply(f)` | `map` | new CF with transformed value |
| `thenCompose(f)` | `flatMap` | new CF (unwrapped) |
| `thenCombine(other, bi)` | zip | new CF when both done |
| `thenAccept(c)` | forEach | `CF<Void>` |
| `exceptionally(f)` | catch | recovers from any earlier failure |
| `handle(bi)` | catch+map | sees both success and failure |
| `*Async` variant | — | runs the stage on a separate thread |

> **Watch out:** without `*Async`, the next stage runs on whatever thread completed the previous one (often the executor that ran the original task). For chains containing slow work, prefer `*Async` with an explicit executor — otherwise you can starve the upstream pool.

### 5.6 Synchronisers: latch, barrier, semaphore

```java
// CountDownLatch — one-shot gate. N threads count down; awaiters proceed when count hits 0.
CountDownLatch ready = new CountDownLatch(3);
// workers:  ready.countDown();
// main:     ready.await();        // unblocks when 3 workers have counted down
//
// NOT reusable — once it's at 0, it stays at 0.

// CyclicBarrier — reusable. N threads wait at a barrier; all proceed together.
CyclicBarrier barrier = new CyclicBarrier(3, () -> log.info("phase done"));
// each thread: barrier.await();   // blocks until 3 threads are waiting
// then all 3 release, barrier resets for the next round.

// Semaphore — controls concurrent access to N permits. Use for rate limiting,
// bounded resource pools (DB connections, file handles).
Semaphore sem = new Semaphore(10);
sem.acquire();
try { useResource(); } finally { sem.release(); }
```

### 5.7 `BlockingQueue` and the producer-consumer pattern

The cleanest way to coordinate producers and consumers — backpressure for free:

```java
BlockingQueue<Trade> queue = new ArrayBlockingQueue<>(1000);

// Producer
class Producer implements Runnable {
    public void run() {
        while (running) {
            queue.put(receiveTrade());     // BLOCKS if queue is full → backpressure
        }
        queue.put(POISON_PILL);            // tell consumers to stop
    }
}

// Consumer
class Consumer implements Runnable {
    public void run() {
        while (true) {
            Trade t = queue.take();        // BLOCKS if queue is empty
            if (t == POISON_PILL) break;
            process(t);
        }
    }
}
```

| Implementation | Properties |
|----------------|-----------|
| `ArrayBlockingQueue` | Bounded, array-backed, optional fairness |
| `LinkedBlockingQueue` | Optionally bounded, higher throughput, more memory churn |
| `PriorityBlockingQueue` | Unbounded, ordered by `Comparator` |
| `SynchronousQueue` | Capacity zero — `put` blocks until a `take` is waiting (direct handoff) |

### 5.8 `ThreadLocal` — per-thread storage (mind the leak)

Each thread sees its own copy. No synchronisation needed:

```java
private static final ThreadLocal<SimpleDateFormat> FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String today() { return FMT.get().format(new Date()); }
```

> ⚠ **Critical in thread pools.** Pool threads get reused. If you `set()` a value and don't `remove()` it, the next task on that thread sees stale data — request context bleeds across requests. **Always pair set with remove in `finally`:**

```java
try {
    RequestContext.set(currentRequest);
    handle();
} finally {
    RequestContext.remove();   // mandatory in pooled environments
}
```

### 5.9 `ForkJoinPool` and work stealing

Designed for **recursive, CPU-bound, divide-and-conquer** tasks. Worker threads "steal" tasks from each other's queues to keep all cores busy.

```java
class SumTask extends RecursiveTask<Long> {
    static final int THRESHOLD = 10_000;
    final int[] arr; final int lo, hi;

    @Override protected Long compute() {
        if (hi - lo < THRESHOLD) {
            long s = 0; for (int i = lo; i < hi; i++) s += arr[i]; return s;
        }
        int mid = (lo + hi) >>> 1;
        SumTask left  = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();                    // submit left to the pool
        long r = right.compute();       // do right in this thread
        return left.join() + r;         // wait for left
    }
}
```

`ForkJoinPool.commonPool()` is shared; it backs parallel streams and any `CompletableFuture.*Async` call without an explicit executor. **Don't use it for blocking I/O** — you'll starve the parallel-stream and CompletableFuture work happening elsewhere.

### 5.10 Virtual threads (Java 21+)

A virtual thread is **scheduled by the JVM, not the OS**. Cheap to create (a few KB instead of ~1 MB), so you can have millions.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            HttpResponse<String> r = http.send(req, BodyHandlers.ofString());
            process(r);
        });
    }
}
```

When the virtual thread hits a blocking operation (HTTP, DB, sleep), the JVM **unmounts** it from its OS *carrier* thread, freeing the carrier to run a different virtual thread. No callback hell, but written like blocking code.

> ⚠ **Two real gotchas:**
> 1. **Don't pool virtual threads** — they're disposable. Pooling them defeats the whole point. `Executors.newVirtualThreadPerTaskExecutor()` is purpose-built.
> 2. **Through Java 21, `synchronized` blocks pin the carrier thread** — the virtual thread can't unmount while inside one. For code likely to block under a `synchronized`, use `ReentrantLock` instead. (Java 24+ removes most of this pinning; check your runtime.)
> 3. **Virtual threads buy you nothing for CPU-bound work.** They shine when threads spend most of their time blocked on I/O.

### 5.11 Deadlock — the four conditions, and how to break them

A deadlock requires **all four** of these to hold simultaneously:

1. **Mutual exclusion** — at least one resource is held in non-shareable mode.
2. **Hold and wait** — a thread holds at least one resource while waiting to acquire another.
3. **No preemption** — resources can't be forcibly taken from a thread.
4. **Circular wait** — there's a cycle of threads each waiting for a resource held by the next.

**Break any one and deadlock is impossible.** The common, practical fix is to break #4 by always acquiring locks in the same global order:

```java
// BAD — opposite orders deadlock under concurrent transfer in opposite directions.
void transfer(Account from, Account to, int amount) {
    synchronized (from) {
        synchronized (to) {
            from.balance -= amount;
            to.balance   += amount;
        }
    }
}

// GOOD — always lock the lower-id account first. Cycle impossible.
void transfer(Account from, Account to, int amount) {
    Account first  = from.id < to.id ? from : to;
    Account second = from.id < to.id ? to   : from;
    synchronized (first) {
        synchronized (second) {
            from.balance -= amount;
            to.balance   += amount;
        }
    }
}

// ALTERNATIVE — break "hold and wait" with tryLock and a timeout.
if (lockA.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        if (lockB.tryLock(100, TimeUnit.MILLISECONDS)) {
            try { /* work */ } finally { lockB.unlock(); }
        } else { /* couldn't get B → release A and retry/back off */ }
    } finally { lockA.unlock(); }
}
```

**Two cousins worth knowing:**
- **Livelock** — threads aren't blocked, but they keep yielding to each other and making no progress (two people in a corridor each stepping aside repeatedly).
- **Starvation** — a thread can't get a lock because faster/greedy threads keep grabbing it. Fix with a fair lock (`new ReentrantLock(true)`).

---

## 6. Memory Aids

### Decision tree: "I have shared state. What do I use?"

```
Is it just one boolean flag, written rarely, read often?
└── volatile boolean

Is it a counter or simple atomic value?
├── Low–medium contention?            → AtomicInteger / AtomicLong / AtomicReference
└── Very high contention (metrics)?   → LongAdder / LongAccumulator

Is it a multi-step compound operation? (e.g. "if absent, put")
├── It's on a Map?                     → ConcurrentHashMap.computeIfAbsent / merge
├── Otherwise simple critical section? → synchronized (block, on a private lock object)
└── Need timeout / fairness / multiple conditions? → ReentrantLock

Is it heavy reads, rare writes? (e.g. config snapshot)
└── ReentrantReadWriteLock or StampedLock (optimistic read)

Producers and consumers?
└── BlockingQueue (ArrayBlockingQueue if bounded)

Async pipeline of dependent steps?
└── CompletableFuture

Many short-lived I/O-bound tasks?
└── Java 21+: virtual threads. Pre-21: an I/O-sized ExecutorService.
```

### "If they ask X, first think Y" mnemonics

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "What does `synchronized` lock?" | The OBJECT, not the method. | Instance method → `this`; static → `Class` object; block → whatever you pass. |
| "Why is `volatile x++` broken?" | Read-add-write is three steps. | volatile makes each read/write visible; doesn't make the trio atomic. |
| "When use `volatile` over `Atomic*`?" | One writer, many readers, no compute. | Flags only. Counters → atomics. |
| "When use `synchronized` over `ReentrantLock`?" | Auto-unlock + simpler. | Use synchronized unless you need timeout, fairness, or multiple conditions. |
| "When CompletableFuture vs Future?" | Composability. | Future = poll/block. CF = chain, combine, recover. |
| "How big should my thread pool be?" | CPU vs I/O. | CPU-bound: cores+1. I/O-bound: cores × (1 + wait/compute). |
| "Why is double-checked locking subtle?" | Reordering of `new`. | Need volatile on the instance field — JMM publishes the reference. |
| "What's a virtual thread?" | JVM-scheduled, cheap, blocking-friendly. | Millions of them; great for I/O; don't pool them; mind synchronized pinning. |

### The three anchor problems (recite in order)

1. **Race condition** = two threads stomp on shared data.
2. **Visibility** = my write sat in a CPU cache, your read missed it.
3. **Deadlock** = circular wait — A holds X waiting for Y, B holds Y waiting for X.

Every concurrency tool maps to one of these. If you can name the problem the question is really about, you can pick the tool.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

> Cover the answer, say it aloud in ≤30 seconds, then check.

### Q1: What does `synchronized` do, and what does it lock?
**A:** Acquires the intrinsic monitor lock on an object. Instance method → locks `this`. Static method → locks the `Class` object. Block → locks whatever object you pass. Reentrant — same thread can re-acquire without blocking.

### Q2: What does `volatile` guarantee? What doesn't it?
**A:** Guarantees **visibility** (every read sees the latest write), **ordering** (happens-before edge), and atomic 64-bit reads/writes. Does NOT guarantee atomicity of compound operations. `volatile int x; x++;` is still a race.

### Q3: Why is `volatile x++` broken?
**A:** `x++` is read → add → write. Volatile makes each read and each write visible, but another thread can squeeze a read between your read and write → lost update.

### Q4: How does `AtomicInteger.incrementAndGet()` work?
**A:** A CAS (compare-and-swap) loop. Read current value, compute new, atomically swap only if the value hasn't changed since the read. If it changed, retry. Lock-free — no thread ever blocks.

### Q5: `synchronized` vs `ReentrantLock` — when each?
**A:** `synchronized` is simpler and auto-unlocks on exception. `ReentrantLock` adds `tryLock` with timeout, interruptible acquisition, fairness, multiple `Condition` variables. Use `synchronized` by default; reach for `ReentrantLock` for those features (or under Java 21+ virtual threads, where `synchronized` can pin the carrier).

### Q6: What's a deadlock, and how do you prevent one?
**A:** Two threads each hold a lock the other needs. Prevent by (1) always acquiring locks in the same global order, (2) using `tryLock` with timeout, (3) keeping critical sections small, (4) avoiding nested locks where possible.

### Q7: Race condition vs livelock vs starvation?
**A:** **Race condition** — outcome depends on interleaving → corruption. **Livelock** — threads not blocked but keep yielding to each other, making no progress. **Starvation** — a thread can't get the lock because others keep grabbing it. Fix starvation with a fair lock.

### Q8: What's the JMM? Define happens-before.
**A:** The Java Memory Model defines visibility and ordering between threads. If A happens-before B, B sees A's effects. Edges: program order, monitor lock unlock → subsequent lock, volatile write → subsequent read, `Thread.start()` → actions in started thread, all actions in thread → return from `join()`, transitivity.

### Q9: `Future` vs `CompletableFuture`?
**A:** `Future` is poll/block — `get()` blocks the caller. `CompletableFuture` composes: `supplyAsync → thenApply → thenCompose → thenCombine → exceptionally`. Non-blocking callbacks, error propagation through the chain.

### Q10: `shutdown()` vs `shutdownNow()` on `ExecutorService`?
**A:** `shutdown()` stops accepting new tasks but lets queued tasks finish. `shutdownNow()` interrupts running tasks and returns the list of unstarted queued tasks. Always pair with `awaitTermination()`.

### Q11: `Runnable` vs `Callable`?
**A:** `Runnable.run()` returns void and can't throw checked exceptions. `Callable<V>.call()` returns `V` and can throw `Exception`. Submit a `Callable` to an `ExecutorService` to get a `Future<V>`.

### Q12: `CountDownLatch` vs `CyclicBarrier` vs `Semaphore`?
**A:** **Latch** — one-shot; N threads count down, waiters proceed at 0; not reusable. **Barrier** — reusable; N threads wait, all proceed together. **Semaphore** — N permits; acquire/release; rate limit or bounded resource pool.

### Q13: `wait`/`notify` vs `Condition`?
**A:** `wait`/`notify` requires `synchronized` on the same monitor — one wait-set per monitor. `Condition` (from `ReentrantLock`) gives you multiple wait-sets per lock — e.g. bounded queue with `notFull` and `notEmpty`. Always wait in a `while`, not `if` (spurious wakeups).

### Q14: What's `ReadWriteLock`?
**A:** Multiple concurrent readers OR one exclusive writer. Use when reads vastly outnumber writes. `StampedLock` (Java 8+) adds optimistic reads — read without locking, validate after — for even better read throughput.

### Q15: What's `ThreadLocal`, and what's the trap?
**A:** Per-thread storage — each thread has its own copy. Common for request context, formatters. Trap: in pooled threads, the value survives between tasks. **Always `remove()` in `finally`.**

### Q16: Why is `ConcurrentHashMap.computeIfAbsent` important?
**A:** Atomic compound operation. Replaces the racy `if (!map.containsKey(k)) map.put(k, v)` check-then-act, where another thread can sneak in between the check and the put.

### Q17: `LongAdder` vs `AtomicLong`?
**A:** `LongAdder` shards the counter across cells, so threads under high contention update different cells and CAS-collide rarely. Sum is eventually consistent — fine for metrics, not for exact accounting.

### Q18: What's a `ForkJoinPool`?
**A:** Work-stealing thread pool for recursive divide-and-conquer CPU work. Backs parallel streams and `CompletableFuture.*Async` (commonPool). Don't use for blocking I/O — it'll starve the pool.

### Q19: Virtual threads — what changes for me?
**A:** JVM-scheduled, cheap (KB not MB), so you can have millions. Block-style code stays simple; the JVM unmounts the virtual thread from its OS carrier when it blocks. Don't pool them. Through Java 21, `synchronized` pins the carrier — prefer `ReentrantLock` for code likely to block under a lock.

### Q20: What's the "double-checked locking" pattern, and what's its catch?
**A:** Cheap fast path (no lock), expensive slow path (lock + recheck) for lazy singletons. Catch: the singleton field **must** be `volatile`, or the JIT can publish a partially-constructed object due to instruction reordering.

---

### Key Code Patterns

**Double-checked locking singleton**
```java
public class Config {
    private static volatile Config instance;     // volatile is REQUIRED
    public static Config getInstance() {
        Config local = instance;                 // single volatile read
        if (local == null) {
            synchronized (Config.class) {
                local = instance;
                if (local == null) instance = local = new Config();
            }
        }
        return local;
    }
}
```

**Thread-safe counter (lock-free)**
```java
private final AtomicLong counter = new AtomicLong();
public long next() { return counter.incrementAndGet(); }
```

**CompletableFuture chain with recovery**
```java
CompletableFuture.supplyAsync(() -> fetchUser(id), pool)
    .thenApply(User::email)
    .thenCompose(this::sendEmailAsync)
    .thenCombine(fetchPrefsAsync(id), (sent, prefs) -> finalise(sent, prefs))
    .exceptionally(ex -> { log.error("failed", ex); return fallback; });
```

**Producer–consumer with `BlockingQueue`**
```java
BlockingQueue<Task> q = new ArrayBlockingQueue<>(100);
// producer:  q.put(task);                // blocks if full
// consumer:  Task t = q.take();          // blocks if empty
```

**Deadlock-safe transfer (consistent lock order)**
```java
Account first  = from.id < to.id ? from : to;
Account second = from.id < to.id ? to   : from;
synchronized (first) {
    synchronized (second) {
        from.balance -= amount;
        to.balance   += amount;
    }
}
```

---

## 8. Self-Test

**Easy** (you should answer cold, no prep)
- [ ] Why does `.run()` not start a new thread, and what should you call instead?
- [ ] What's the difference between a thread and a process?
- [ ] What does `Thread.join()` do?
- [ ] Name the three big problems in concurrency.

**Medium**
- [ ] Walk me through the lost-update example from §2 step by step.
- [ ] Explain why `volatile int x; x++;` is still a race condition.
- [ ] What does `synchronized` actually lock, and what's the difference between locking on `this` vs on `Account.class`?
- [ ] When would you reach for `ReentrantLock` over `synchronized`?
- [ ] How would you size a thread pool for an HTTP-fetching worker?

**Hard**
- [ ] Define "happens-before" and give two examples of how it gets established.
- [ ] Why does double-checked locking need `volatile`?
- [ ] Explain the four conditions for deadlock and how to break each.
- [ ] How does `LongAdder` outperform `AtomicLong` under high contention, and what's the trade-off?
- [ ] What pins a virtual thread to its carrier, and why does it matter?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Thread** | A worker that runs your code. One JVM has many. |
| **Process** | A separately-running program. Threads share memory; processes don't. |
| **Race condition** | Two threads stomp on shared data → wrong result. |
| **Critical section** | The code that must run uninterrupted to be correct. |
| **Lock / monitor / mutex** | A "door handle" only one thread can hold. |
| **Reentrant lock** | A lock that the same thread can grab again without blocking itself. |
| **`synchronized`** | Java's built-in lock on an object's intrinsic monitor. |
| **`volatile`** | "Always go to main memory for this variable" — visibility, no atomicity. |
| **Atomic operation** | Done in one indivisible step. No other thread can see it half-finished. |
| **CAS (compare-and-swap)** | A CPU instruction that swaps a value only if it still matches the expected value. |
| **JMM (Java Memory Model)** | The spec saying when one thread's writes become visible to another. |
| **Happens-before** | A guarantee that if A happens-before B, B sees A's effects. |
| **Deadlock** | Two+ threads each waiting for a resource the other holds → freeze. |
| **Livelock** | Threads aren't blocked but keep deferring to each other → no progress. |
| **Starvation** | A thread never gets the lock because others keep taking it. |
| **Daemon thread** | A background thread the JVM is willing to abandon at exit. |
| **Thread pool** | A team of worker threads that process tasks from a queue. |
| **Backpressure** | Slowing producers down when consumers can't keep up (e.g. bounded queue blocks `put`). |
| **Carrier thread** | The OS thread that runs a virtual thread (Java 21+). |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/01_concurrency_doubts.md) · [Next: 02 Collections →](./02_collections.md)

[↑ Back to top](#01--concurrency--threading)
