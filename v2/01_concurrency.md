# 01 — Concurrency & Threading

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## 🟢 Start Here — Concurrency in Plain English

### What is concurrency?

Imagine a **restaurant kitchen**. One chef (single thread) can only do one thing at a time — chop, then cook, then plate. But a busy restaurant has **multiple chefs** (multiple threads) working at the same time. That's concurrency: multiple tasks making progress at the same time.

In Java, your program runs on **threads**. By default you have one (the "main" thread). When you create more threads, they can do work simultaneously — like having more chefs in the kitchen.

### Why is it tricky?

The problem starts when two chefs reach for the **same knife** at the same time. In programming, that "knife" is **shared data** — a variable, a list, a counter that multiple threads read and write.

```java
// Imagine two threads both running this at the same time:
int balance = 1000;

// Thread A: withdraw 500         // Thread B: withdraw 500
// A reads balance: 1000          // B reads balance: 1000  (BEFORE A writes!)
// A sets balance: 500            // B sets balance: 500    (B didn't see A's change!)
// Result: balance = 500, but we withdrew 1000 total!
// This is called a RACE CONDITION — the outcome depends on who runs first.
```

### How do we fix it?

**Locks.** Think of a lock like a bathroom door lock — only one person can enter at a time. Everyone else waits.

```java
// `synchronized` is Java's simplest lock:
synchronized (this) {
    // Only ONE thread can be inside this block at a time.
    // Other threads wait at the door until the first one exits.
    balance -= amount;
}
// Now both withdrawals happen one at a time — no lost money.
```

### Key vocabulary (in simple terms)

| Word | What it means (simply) |
|------|----------------------|
| **Thread** | A worker that runs your code. You can have many workers. |
| **Race condition** | Two workers clash over the same thing → wrong result. |
| **Lock / synchronized** | A door lock — only one worker enters at a time. |
| **Deadlock** | Two workers each waiting for the other to finish — nobody moves. |
| **volatile** | A "shared whiteboard" — when one worker writes, all others can immediately read the new value. |
| **Atomic** | An operation that's impossible to interrupt halfway. Like flipping a light switch — it's either on or off, never halfway. |
| **Thread pool** | A team of workers standing ready. You give them tasks instead of hiring a new worker for every task. |

### The three big problems in concurrency

1. **Race conditions** — two threads change the same data → corruption. Fix: locks, atomics.
2. **Deadlocks** — two threads each hold what the other needs → freeze. Fix: always lock in the same order.
3. **Visibility** — one thread changes a value but the other thread doesn't see the change (stale data). Fix: `volatile` or `synchronized`.

> Once you understand these three problems, the rest of concurrency is just different tools to solve them more efficiently.

---

## 📚 Study Material

### 1. Thread Lifecycle

A Java thread moves through these states (defined in `Thread.State`):

```
NEW → RUNNABLE → (BLOCKED | WAITING | TIMED_WAITING) → TERMINATED
```

| State | How you get here | How you leave |
|-------|-----------------|---------------|
| `NEW` | `new Thread()` — not yet started | `thread.start()` |
| `RUNNABLE` | `start()` called — OS may or may not be running it | Completes, or enters blocked/waiting |
| `BLOCKED` | Waiting to acquire a `synchronized` monitor | Lock becomes available |
| `WAITING` | `wait()`, `join()`, `LockSupport.park()` | `notify()`, thread completes, `unpark()` |
| `TIMED_WAITING` | `sleep(ms)`, `wait(ms)`, `join(ms)` | Timeout expires or notification |
| `TERMINATED` | `run()` returns or throws uncaught exception | — |

💡 **Interview tip:** `RUNNABLE` means *eligible* to run — the OS scheduler decides when it actually gets CPU time. There's no `RUNNING` state in Java's model.

### 2. Creating Threads — Three Ways

```java
// Way 1: Extend Thread (rarely used — wastes inheritance)
class MyThread extends Thread {
    @Override
    public void run() { System.out.println("Running"); }
}
new MyThread().start();

// Way 2: Implement Runnable (preferred before Java 8)
Thread t = new Thread(() -> System.out.println("Running"));
t.start();

// Way 3: Callable + Future (when you need a return value)
ExecutorService exec = Executors.newFixedThreadPool(4);
Future<String> future = exec.submit(() -> {   // Callable<String>
    return "result";                           // can throw checked exceptions
});
String result = future.get();                  // blocks until done
```

⚠️ **Never call `run()` directly** — that executes on the caller's thread. Only `start()` creates a new OS thread.

### 3. `synchronized` — Deep Dive

Every Java object has an **intrinsic monitor lock** (mutex). `synchronized` acquires it.

```java
// Locks on `this` — the instance
public synchronized void transfer(Account to, int amount) {
    this.balance -= amount;     // only one thread at a time per instance
    to.deposit(amount);
}

// Locks on the Class object — shared across ALL instances
public static synchronized Config getConfig() { ... }

// Block form — lock on specific object (finer granularity)
public void update() {
    synchronized (lockObject) {       // only threads contending on SAME lockObject block
        // critical section
    }
}
```

**How it works internally:**
1. Thread attempts to acquire the object's monitor
2. If free → thread takes ownership, enters critical section
3. If held by another thread → calling thread enters `BLOCKED` state
4. On exit (normal or exception) → monitor is released automatically

**Reentrant:** Same thread can re-acquire a lock it already holds (counter increments). This prevents self-deadlock:

```java
public synchronized void a() { b(); }     // already holds lock on `this`
public synchronized void b() { /* OK */ }  // re-enters — counter goes to 2
```

### 4. `volatile` — Visibility, Not Atomicity

```java
// Thread A                          // Thread B
volatile boolean running = true;
                                     // Without volatile, Thread B may NEVER see
while (running) {                    // the update — it reads a cached/stale copy
    doWork();                        // from its CPU cache or register
}
// Thread A: running = false;        // volatile forces a memory barrier:
                                     // write flushes to main memory,
                                     // read always fetches from main memory
```

**What volatile guarantees:**
- **Visibility:** All threads see the latest write
- **Ordering:** Establishes a *happens-before* edge (no reordering across the volatile access)

**What volatile does NOT guarantee:**
- **Atomicity of compound operations**

```java
volatile int count = 0;
count++;  // THIS IS STILL BROKEN — three steps:
          // 1. READ count (volatile read — sees latest)
          // 2. ADD 1 (local computation)
          // 3. WRITE count (volatile write — publishes)
          // Another thread can READ between steps 1 and 3 → lost update
```

### 5. Java Memory Model (JMM) & Happens-Before

The JMM defines **when one thread's writes become visible to another thread**. Without a happens-before relationship, the JVM/CPU can reorder instructions and cache values — you see stale data.

**Happens-before rules (memorise these):**

| Rule | What it means |
|------|--------------|
| Program order | Within a single thread, each statement happens-before the next |
| Monitor lock | `synchronized` unlock → happens-before → subsequent lock on SAME monitor |
| Volatile | Volatile write → happens-before → subsequent volatile read of SAME field |
| Thread start | `thread.start()` → happens-before → any action in the started thread |
| Thread join | All actions in a thread → happen-before → `join()` returns |
| Transitivity | If A hb B and B hb C, then A hb C |

⚠️ **Why this matters:** Without happens-before, `double-checked locking` without `volatile` is broken — another thread can see a partially constructed object (instruction reordering).

### 6. CAS (Compare-And-Swap) & Atomics

CAS is a **CPU instruction** that atomically does: "If the current value equals the expected value, set it to the new value; otherwise do nothing and return the current value."

```java
// AtomicInteger uses CAS internally — no locks, no blocking
AtomicInteger counter = new AtomicInteger(0);

// incrementAndGet() internally does:
// do {
//     int current = get();             // read
//     int next = current + 1;          // compute
// } while (!compareAndSet(current, next));  // CAS — retry if someone else changed it
//
// This is lock-free: no thread ever blocks. Under LOW contention, very fast.
// Under HIGH contention, many retries → consider LongAdder instead.

counter.incrementAndGet();     // atomic: returns new value
counter.compareAndSet(5, 10);  // if current == 5, set to 10, return true

// LongAdder — sharded counter for high contention
LongAdder adder = new LongAdder();  // internally uses Cell[] array
adder.increment();                   // threads update different cells — fewer CAS collisions
adder.sum();                         // eventually consistent — sums all cells
```

**When to use what:**

| Scenario | Use |
|----------|-----|
| Simple counter, low contention | `AtomicInteger` / `AtomicLong` |
| Counter, high contention (metrics) | `LongAdder` |
| Atomic reference swap | `AtomicReference<T>` |
| Atomic field update (memory-efficient) | `AtomicIntegerFieldUpdater` |

### 7. Lock Hierarchy — Beyond `synchronized`

```java
// ReentrantLock — explicit locking with more features
ReentrantLock lock = new ReentrantLock(true);  // fair=true → FIFO ordering (slower)
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();  // YOU must unlock — synchronized does this automatically
}

// tryLock — non-blocking attempt (prevents deadlock)
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try { /* work */ } finally { lock.unlock(); }
} else {
    // couldn't acquire — take alternative action
}

// Condition — multiple wait-sets on one lock (wait/notify on steroids)
Condition notFull  = lock.newCondition();
Condition notEmpty = lock.newCondition();
// Producer: while (count == MAX) notFull.await();  ... notEmpty.signal();
// Consumer: while (count == 0) notEmpty.await();   ... notFull.signal();
```

```java
// ReadWriteLock — multiple readers OR one writer
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();    // multiple threads can hold this simultaneously
rwLock.writeLock().lock();   // exclusive — blocks readers and other writers

// StampedLock (Java 8+) — optimistic reads for even better read throughput
StampedLock sl = new StampedLock();
long stamp = sl.tryOptimisticRead();     // no lock acquired — just a stamp
double x = this.x, y = this.y;          // read fields
if (!sl.validate(stamp)) {              // check if a write happened since stamp
    stamp = sl.readLock();               // fall back to pessimistic read
    try { x = this.x; y = this.y; } finally { sl.unlockRead(stamp); }
}
```

### 8. ExecutorService & Thread Pool Sizing

**Never create threads manually in production.** Use a thread pool.

```java
// Fixed pool — known concurrency level
ExecutorService fixed = Executors.newFixedThreadPool(8);

// Cached pool — grows/shrinks (dangerous: unbounded)
ExecutorService cached = Executors.newCachedThreadPool();

// Scheduled — delayed/periodic tasks
ScheduledExecutorService sched = Executors.newScheduledThreadPool(4);
sched.scheduleAtFixedRate(() -> cleanup(), 0, 1, TimeUnit.MINUTES);

// Single thread — sequential task processing (guaranteed order)
ExecutorService single = Executors.newSingleThreadExecutor();

// PRODUCTION: Use ThreadPoolExecutor directly for control
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4,                                   // corePoolSize
    16,                                  // maxPoolSize
    60, TimeUnit.SECONDS,               // idle thread keepalive
    new ArrayBlockingQueue<>(1000),      // bounded queue (IMPORTANT)
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
);
```

**Thread pool sizing formula:**

| Workload | Formula | Why |
|----------|---------|-----|
| CPU-bound | `cores + 1` | Extra thread covers page faults; more threads = context-switch waste |
| I/O-bound | `cores × (1 + waitTime/computeTime)` | Threads spend most time waiting; more threads keep CPU busy |
| Mixed | Separate pools for CPU vs I/O work | Prevents I/O tasks from starving CPU tasks |

⚠️ **Never use `Executors.newCachedThreadPool()` in production** — under burst load it creates unlimited threads → OOM.

### 9. CompletableFuture — Async Composition

```java
// Basic async pipeline
CompletableFuture.supplyAsync(() -> fetchUser(id), executor)  // runs on executor
    .thenApply(user -> user.getEmail())          // sync transform (same thread)
    .thenApplyAsync(email -> validate(email))    // async transform (ForkJoinPool)
    .thenCompose(email -> sendEmailAsync(email)) // flatMap — unwraps inner CF
    .thenAccept(result -> log.info("Sent"))      // terminal — consumes result
    .exceptionally(ex -> {                       // catches ANY exception in chain
        log.error("Failed", ex);
        return fallback;
    });

// Combine two independent futures
CompletableFuture<User> userCf = fetchUserAsync(id);
CompletableFuture<Prefs> prefsCf = fetchPrefsAsync(id);

CompletableFuture<Profile> profile = userCf
    .thenCombine(prefsCf, (user, prefs) -> merge(user, prefs));  // runs when BOTH complete

// Wait for all / any
CompletableFuture.allOf(cf1, cf2, cf3).join();   // blocks until ALL complete
CompletableFuture.anyOf(cf1, cf2, cf3).join();   // blocks until FIRST completes
```

**Key distinctions:**

| Method | Behaviour |
|--------|-----------|
| `thenApply` | Transform result (like `map`) |
| `thenCompose` | Transform to another CF (like `flatMap`) |
| `thenCombine` | Combine two independent CFs |
| `thenAccept` | Consume result, return void |
| `exceptionally` | Handle exception, return recovery value |
| `handle` | Handle BOTH success and failure |
| `*Async` variants | Run on a different thread (default: ForkJoinPool.commonPool) |

### 10. Synchronisers

```java
// CountDownLatch — one-shot gate: N threads count down, waiters proceed at 0
CountDownLatch latch = new CountDownLatch(3);
// Worker threads: latch.countDown();     // each worker decrements
// Main thread:    latch.await();         // blocks until count == 0
// NOT reusable — once triggered, stays at 0 forever

// CyclicBarrier — reusable: N threads wait, ALL proceed together
CyclicBarrier barrier = new CyclicBarrier(3, () -> log.info("Phase done"));
// Each thread:    barrier.await();       // blocks until 3 threads are waiting
// Then all 3 proceed — barrier resets for next round

// Semaphore — controls concurrent access to N permits
Semaphore sem = new Semaphore(10);        // 10 concurrent permits
sem.acquire();                            // blocks if no permits available
try { accessResource(); } finally { sem.release(); }
// Use for: connection pools, rate limiting, resource throttling

// Phaser (Java 7+) — flexible barrier with dynamic registration
Phaser phaser = new Phaser(3);            // 3 parties
phaser.arriveAndAwaitAdvance();           // like CyclicBarrier but parties can join/leave
```

### 11. ThreadLocal — Per-Thread Isolation

```java
// Each thread gets its own copy — no synchronisation needed
private static final ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String format(Date d) {
    return dateFormat.get().format(d);  // thread-safe: each thread has its own formatter
}
```

⚠️ **Critical in thread pools:** ThreadLocal values survive between tasks because the thread is reused. **Always clean up:**

```java
try {
    RequestContext.set(context);
    processRequest();
} finally {
    RequestContext.remove();  // MUST remove — otherwise context leaks to next request
}
```

### 12. ForkJoinPool — Work Stealing

Designed for **recursive, CPU-bound** divide-and-conquer tasks.

```java
class SumTask extends RecursiveTask<Long> {
    private final int[] arr;
    private final int lo, hi;
    
    @Override
    protected Long compute() {
        if (hi - lo < THRESHOLD) {            // base case — compute directly
            long sum = 0;
            for (int i = lo; i < hi; i++) sum += arr[i];
            return sum;
        }
        int mid = (lo + hi) / 2;
        SumTask left = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();                           // submit left to pool
        long rightResult = right.compute();    // compute right in current thread
        long leftResult = left.join();         // wait for left
        return leftResult + rightResult;
    }
}

// ForkJoinPool.commonPool() — shared pool sized to availableProcessors()
// Backs: parallel streams, CompletableFuture.supplyAsync() (no executor arg)
// ⚠️ Don't use for I/O — blocking tasks starve CPU-bound work
```

### 13. Deadlock — The Four Conditions (ALL must hold)

1. **Mutual exclusion** — resource held exclusively
2. **Hold and wait** — thread holds one lock while waiting for another
3. **No preemption** — locks can't be forcibly taken
4. **Circular wait** — A waits for B, B waits for A

**Break ANY ONE to prevent deadlock:**

```java
// DEADLOCK — threads lock in different order
// Thread 1: lock(A) → lock(B)
// Thread 2: lock(B) → lock(A)

// FIX: Always lock in the same global order
void transfer(Account from, Account to, int amount) {
    Account first  = from.id < to.id ? from : to;   // deterministic order
    Account second = from.id < to.id ? to : from;
    synchronized (first) {
        synchronized (second) {
            from.balance -= amount;
            to.balance += amount;
        }
    }
}

// FIX 2: tryLock with timeout
if (lockA.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        if (lockB.tryLock(100, TimeUnit.MILLISECONDS)) {
            try { /* work */ } finally { lockB.unlock(); }
        }
    } finally { lockA.unlock(); }
}
```

### 14. Virtual Threads (Java 21+)

```java
// Platform thread = OS thread (expensive: ~1MB stack, kernel-scheduled)
// Virtual thread = lightweight, JVM-scheduled, runs on carrier platform threads

// Create millions of virtual threads — no pool sizing needed
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            // I/O-bound work is perfect — virtual thread parks cheaply
            var result = httpClient.send(request, BodyHandlers.ofString());
            process(result);
        });
    }
}

// ⚠️ Don't use for CPU-bound work — no benefit over platform threads
// ⚠️ Don't pool virtual threads — they're disposable (create per task)
// ⚠️ synchronized blocks pin the carrier thread — prefer ReentrantLock
```

### 15. Producer-Consumer Pattern (Full Example)

```java
// BlockingQueue handles all synchronisation for you
BlockingQueue<Trade> queue = new ArrayBlockingQueue<>(1000);

// Producer thread
class TradeProducer implements Runnable {
    public void run() {
        while (running) {
            Trade trade = receiveTrade();
            queue.put(trade);              // blocks if queue is full (backpressure)
        }
        queue.put(POISON_PILL);            // signal consumers to stop
    }
}

// Consumer thread
class TradeProcessor implements Runnable {
    public void run() {
        while (true) {
            Trade trade = queue.take();    // blocks if queue is empty
            if (trade == POISON_PILL) break;
            process(trade);
        }
    }
}

// BlockingQueue implementations:
// ArrayBlockingQueue  — bounded, array-backed, fair option
// LinkedBlockingQueue — optionally bounded, higher throughput than ABQ
// PriorityBlockingQueue — unbounded, priority-ordered
// SynchronousQueue    — zero capacity: put blocks until take (direct handoff)
```

---

## Rapid-Fire Q&A (drill these aloud, 30 sec each)

### Q1: What does `synchronized` do and what does it lock?
**A:** Acquires the intrinsic monitor lock on an object. Instance method → locks `this`. Static method → locks the `Class` object. Block → locks the specified object. It's reentrant — same thread can re-acquire without deadlocking.

### Q2: What does `volatile` guarantee? What doesn't it guarantee?
**A:** Guarantees **visibility** — writes by one thread are immediately visible to others. Establishes a happens-before edge. Does **not** guarantee atomicity. `volatile int x; x++` is a read-modify-write — still a race condition.

### Q3: Why is `volatile int x; x++` still a race condition?
**A:** `x++` is three operations: read x, add 1, write x. Another thread can read between the read and write, causing a lost update. `volatile` only guarantees each individual read/write is visible — not that the compound operation is atomic.

### Q4: What does `AtomicInteger.incrementAndGet()` do internally?
**A:** Uses a CPU-level CAS (compare-and-swap) loop. Read current value, compute new value, atomically write only if the value hasn't changed since the read. If it changed, retry. Lock-free — no thread ever blocks.

### Q5: `synchronized` vs `ReentrantLock` — when use which?
**A:** `ReentrantLock` adds: `tryLock(timeout)`, interruptible acquisition, fair ordering, multiple `Condition` objects. Cost: you must `unlock()` in a `finally` block — `synchronized` auto-unlocks on exception. Use `synchronized` for simple cases; `ReentrantLock` when you need timeout or fairness.

### Q6: What's a deadlock? How do you prevent one?
**A:** Two+ threads each hold a lock the other needs. Prevention: (1) always acquire locks in the same global order, (2) use `tryLock` with timeout, (3) minimize lock scope, (4) avoid nested locking.

### Q7: Race condition vs livelock vs starvation?
**A:** Race condition: outcome depends on thread interleaving — data corruption. Livelock: threads aren't blocked but keep yielding to each other, making no progress. Starvation: a thread can't get the lock because others keep grabbing it — fix with fair locking.

### Q8: What's the Java Memory Model? Define happens-before.
**A:** The JMM defines visibility and ordering guarantees between threads. If action A happens-before action B, then B sees A's effects. Established by: program order within a thread, `synchronized` unlock → lock on same monitor, `volatile` write → read, `Thread.start()` → actions in started thread, thread actions → `join()` return.

### Q9: `Future` vs `CompletableFuture`?
**A:** `Future` is poll/block-based — `get()` blocks the caller. `CompletableFuture` is composable: `supplyAsync → thenApply → thenCompose → exceptionally`. Supports chaining, combining (`thenCombine`, `allOf`, `anyOf`), and non-blocking callbacks.

### Q10: What does `ExecutorService.shutdown()` do vs `shutdownNow()`?
**A:** `shutdown()` stops accepting new tasks, lets queued tasks finish. `shutdownNow()` interrupts running tasks and returns the queue of unstarted tasks.

### Q11: `Runnable` vs `Callable`?
**A:** `Runnable.run()` returns `void`, can't throw checked exceptions. `Callable.call()` returns a value `T` and can throw `Exception`. Submit a `Callable` to get a `Future<T>`.

### Q12: `CountDownLatch` vs `CyclicBarrier` vs `Semaphore`?
**A:** `CountDownLatch`: one-shot — N threads count down, waiting threads proceed when count hits 0. `CyclicBarrier`: reusable — N threads wait at a barrier point, all proceed together. `Semaphore`: controls concurrent access to N permits — acquire/release. Use for rate limiting, connection pools.

### Q13: What's `wait/notify` vs `Condition`?
**A:** `wait/notify` requires `synchronized` on the same monitor — one condition per lock. `Condition` (from `ReentrantLock`) allows multiple conditions per lock. E.g., a bounded queue can have `notFull` and `notEmpty` conditions on the same lock. Always call `wait` in a `while` loop (spurious wakeups).

### Q14: What's `ReadWriteLock`?
**A:** Allows multiple concurrent readers OR one exclusive writer. Use when reads vastly outnumber writes. `StampedLock` (Java 8+) adds optimistic reads for even better read performance.

### Q15: What's `ThreadLocal`?
**A:** Per-thread storage — each thread gets its own copy. Common for request context, database connections, date formatters. **Must** remove in `finally` in thread pool environments — otherwise context leaks between requests.

### Q16: What's `ConcurrentHashMap.computeIfAbsent` and why is it important?
**A:** Atomic compound operation: check if key exists, compute and insert if absent — all in one atomic step. Replaces the racy `if (!map.containsKey(k)) map.put(k, v)` check-then-act pattern.

### Q17: What's `LongAdder` and when is it better than `AtomicLong`?
**A:** Shards the counter across cells to reduce CAS contention. Under high contention, `LongAdder` is significantly faster than `AtomicLong` because threads update different cells. Trade-off: `sum()` is eventually consistent — okay for metrics, not for exact counts.

### Q18: What's a `ForkJoinPool`?
**A:** Work-stealing thread pool. Divide tasks recursively (`fork`), combine results (`join`). Backing pool for parallel streams and `CompletableFuture.supplyAsync()` (common pool). Don't use for I/O — it's sized for CPU-bound work (`Runtime.availableProcessors()`).

---

## Key Code Patterns

### Double-Checked Locking Singleton
```java
public class Config {
    private static volatile Config instance;
    public static Config getInstance() {
        if (instance == null) {                    // 1st check — no lock
            synchronized (Config.class) {
                if (instance == null) {             // 2nd check — under lock
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}
// volatile prevents instruction reordering — without it,
// another thread could see a partially constructed Config
```

### Thread-safe counter
```java
private final AtomicLong counter = new AtomicLong(0);
public long increment() { return counter.incrementAndGet(); } // CAS loop, lock-free
```

### CompletableFuture chain
```java
CompletableFuture.supplyAsync(() -> fetchUser(id), executor)
    .thenApply(user -> user.getEmail())
    .thenCompose(email -> sendEmailAsync(email))    // flatMap
    .thenCombine(fetchPrefs(id), (result, prefs) -> merge(result, prefs))
    .exceptionally(ex -> { log.error("Failed", ex); return fallback; });
```

### Producer-Consumer with BlockingQueue
```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);
// Producer
queue.put(task);           // blocks if full
// Consumer
Task t = queue.take();     // blocks if empty
```

---

## Can you answer these cold?

- [ ] Explain `synchronized` — what it locks, why it's reentrant
- [ ] `volatile` guarantees vs limitations — give the `x++` example
- [ ] CAS — what it is, where Java uses it
- [ ] Four deadlock conditions and how to break each one
- [ ] `Future` vs `CompletableFuture` — write a chain from memory
- [ ] `ExecutorService` factory methods — when each is appropriate
- [ ] `wait/notify` — why `while` not `if` (spurious wakeups)
- [ ] Thread pool sizing: CPU-bound = cores+1, I/O-bound = cores × (1 + wait/compute)

[← Back to Index](./00_INDEX.md)
