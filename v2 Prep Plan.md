# Citi/Karat — 4-Day Prep Plan (v2: Fundamentals + Debugging Focus)

**This supersedes v1.** Same 4 days, completely different weighting based on updated intel. Read this end-to-end once, then operate from it.

## Table of Contents

1. [What Changed and Why](#1-what-changed-and-why)
2. [Karat Format — Updated Picture](#2-karat-format--updated-picture)
3. [The New Skill Mix](#3-the-new-skill-mix)
4. [Non-Negotiables (Adapted)](#4-non-negotiables-adapted)
5. [Day-by-Day Plan](#5-day-by-day-plan)
6. [UMPIRE for Debugging](#6-umpire-for-debugging)
7. [Three Talk Tracks](#7-three-talk-tracks)
8. [Java Fundamentals — Expanded](#8-java-fundamentals--expanded)
9. [Debugging Methodology](#9-debugging-methodology)
10. [Code Extension Methodology](#10-code-extension-methodology)
11. [DSA Insurance (Light Touch)](#11-dsa-insurance-light-touch)
12. [Day-of-Interview Playbook](#12-day-of-interview-playbook)
13. [What's Cut Now](#13-whats-cut-now)

---

## 1. What Changed and Why

The new intel reshapes the interview from "DSA grind" to "Java fluency + code-review skills." Specifically:

- **Concurrency, async, `HashMap` vs `ConcurrentHashMap`-style fundamentals** — front-loaded, rapid-fire
- **Dry-run code, find the syntax error or bug, fix the failing test** — middle of the interview
- **Add another function/method** to the existing code — toward the end
- **Implication**: this is a screening, not a final round. The bar is "is this a real senior Java engineer who can read and reason about code" — not "can they invent a heap-based solution under pressure"

**What this means strategically:**

For an 8-YOE Java engineer who's been doing exactly this kind of work at TD on Enterprise Protect, the format actually plays to your strengths. Reading and debugging real codebases is what you do every day. The skill the interview tests is something you already have — you just need to do it **out loud, with the right vocabulary, under a clock**.

**One important hedge**: the intel is from Python devs. Karat templates vary by client and even by track. The format may shift slightly for Java candidates, and a small algorithmic problem isn't impossible. So we keep a thin DSA insurance layer (Section 11) but stop optimizing for it.

**Old weighting**: 70% DSA, 25% Java fundamentals, 5% communication.
**New weighting**: 40% Java fundamentals (concurrency-heavy), 30% debugging mechanics, 15% extension/design articulation, 10% DSA insurance, plus communication threading through everything.

---

## 2. Karat Format — Updated Picture

| Section | Duration | What happens |
|---|---|---|
| Intro | ~3 min | One-line background. Don't over-talk. |
| **Java fundamentals (rapid-fire)** | **~10–15 min** | Concurrency, collections internals, OOP. They want crisp 1–3 sentence answers with correct vocabulary. Sets the tone. |
| **Debugging problem** | **~20–30 min** | They share existing code with bugs or failing tests. Read it, dry-run it, locate the bug, fix it. Talk through your reasoning the whole time. |
| **Extension problem** | **~15–25 min** | Add a new method to the existing class. Match the existing style. Justify your design choices. |
| Wrap | ~2 min | "Any questions?" — have one ready. |

**What's recorded**: everything — screen, video, audio. A separate Citi engineer reviews the recording. The interviewer doesn't make the hiring decision. **You're performing for the recording.**

**The Karat editor**: plain-text browser editor. No Copilot, no IntelliJ. Practice in any plain-text editor — even Notepad would work for syntax drills.

**What they score (reweighted toward what this format actually tests):**

- **Vocabulary fluency** — do you say "monitor lock" not "lock thingy"? "CAS" not "atomic-ish"? "Race condition," "happens-before," "memory visibility" — the actual technical terms?
- **Reading comprehension** — can you state what existing code does before touching it?
- **Bug isolation** — do you trace systematically or stare and guess?
- **Code quality on extension** — does your new method match existing style, handle the same edges, reuse helpers?
- **Communication throughout** — were you talking the whole time?

**Highest-leverage insight**: At 8 YOE you almost certainly know the answer. The question is whether you can say it crisply with the right terminology. Most of your prep is **rehearsing how to articulate things you already know** — not learning new things.

---

## 3. The New Skill Mix

Three capabilities to build over 4 days:

### Capability 1: Verbal fluency on Java fundamentals (highest priority)

Be able to answer these in 10–30 seconds, with correct jargon:

- "What's the difference between `HashMap`, `Hashtable`, `ConcurrentHashMap`, and `Collections.synchronizedMap`?"
- "Explain `volatile`. Why isn't it enough for a counter?"
- "What does `synchronized` actually lock?"
- "What's a happens-before relationship?"
- "When would you use `CompletableFuture` over `Future`?"
- "Walk me through what happens when I `put` into a `HashMap` and there's already 8 entries in the bucket."

The key skill isn't knowing — it's **saying it without um-ing**. You'll know when you've drilled enough: the sentences come out as a single fluid block, not a hesitant reconstruction.

### Capability 2: Reading and debugging code out loud

Given a 30–80 line Java class with a failing test:

1. State what the class is supposed to do (read first, judge later)
2. Walk through the failing test input step by step, voicing what each line evaluates to
3. Identify where actual diverges from expected
4. Propose a fix and explain why it's correct
5. State whether the fix could break other tests

The dry-run is the load-bearing skill. You need to be comfortable narrating: "Okay so we enter the loop with `i = 0`, `count = 0`, `sum = 0`. First iteration: we read `arr[0]` which is `5`. The condition `arr[i] > threshold` — threshold is 3, so true. We increment count to 1. Add 5 to sum, sum is now 5..."

This is the part most people skip and lose points on.

### Capability 3: Extending existing code in matching style

Given an existing class and a request to add a method:

1. Read the whole class first (don't just jump to writing)
2. Note conventions: naming, error handling, null checks, helper usage
3. State the contract of the new method before coding it
4. Write it in the same style as what's already there
5. Mention what tests you'd add

The trap is treating it like a greenfield problem. The interviewer is checking whether you can integrate into an existing codebase — exactly what you do at Deloitte every engagement.

---

## 4. Non-Negotiables (Adapted)

### Rule 1: No Copilot. No AI. No "view solution."

For 4 days, Copilot stays off. The reflex you need is reading code without autocomplete shouting at you. If you let Copilot autocomplete during prep, the interview editor's dead silence will throw you.

The only allowed lookups: official Java docs, Baeldung for syntax verification, this document.

### Rule 2: Talk out loud. Always.

Every drill. Every dry-run. Every fundamentals question. Even alone in your apartment. Especially alone — that's when the muscle forms.

Record at least one drill per day on your phone or via Zoom-to-self. Listen back once. By Day 4 the cringe drops noticeably.

### Rule 3: Vocabulary first, code second

When you read a fundamentals answer in Section 8 and think "yeah I know that" — don't move on. **Say it out loud, complete sentences, in 15 seconds or less.** If you can't, you don't have it. Knowing and articulating are different skills.

### Rule 4: After every drill, write the one-liner

After every fundamentals question: write one line in your notes — "Question → answer in my words." After every debugging exercise: "Bug type → signal that gave it away → fix pattern."

By Day 4 you'll have ~60 entries. This is your interview-morning review sheet.

---

## 5. Day-by-Day Plan

~5–6 hours/day of focused work. 60-min break in the middle. Sleep is when this consolidates.

### Day 1: Concurrency Fluency + Collections Internals

**Theme**: build the highest-frequency, highest-jargon-density area first — concurrency. The intel says it's front-loaded in the interview; we front-load it in prep.

**Morning Block (2.5 hrs) — CONCURRENCY**

Read Section 8.1 (Concurrency) once, slowly, no laptop.

Then drill these aloud in front of a mirror or recording, 30 seconds each, no notes:

1. What is `synchronized` and what does it lock?
2. Difference between `synchronized` method and `synchronized` block?
3. What does `volatile` guarantee? What doesn't it guarantee?
4. Why is `volatile int x; x++;` still a race condition?
5. What does `AtomicInteger.incrementAndGet()` do internally? (Answer: CAS loop.)
6. `synchronized` vs `ReentrantLock` — when use which?
7. What's a deadlock? How do you prevent one?
8. Race condition vs livelock vs starvation?
9. `wait/notify` vs `Condition` — what's the difference?
10. What's the Java Memory Model? Define happens-before.
11. `Future` vs `CompletableFuture`?
12. What does `ExecutorService.shutdown()` do vs `shutdownNow()`?
13. What's the difference between `Runnable` and `Callable`?
14. When would you use `CountDownLatch` vs `CyclicBarrier` vs `Semaphore`?

If you can't answer all of these crisply by lunch, you're not done with the morning block. **Crisp** = correct jargon, no hedging, complete sentence.

Then write 5 short Java snippets from scratch (no IDE):

```java
// 1. Counter-safe singleton (double-checked locking)
// 2. Thread-safe lazy init using volatile
// 3. AtomicLong-backed counter with getAndIncrement
// 4. ConcurrentHashMap.computeIfAbsent for thread-safe cache
// 5. CompletableFuture chain: supplyAsync → thenApply → thenCompose
```

Verify each works by tracing it mentally. The point isn't memorization — it's recall under no-IDE conditions.

**Afternoon Block (2 hrs) — COLLECTIONS INTERNALS**

Read Section 8.2 (Collections). Then drill aloud:

1. How does `HashMap` work internally? (Buckets, hashing, mod, collision chain, treeification at 8.)
2. `HashMap` vs `Hashtable` vs `ConcurrentHashMap` vs `Collections.synchronizedMap` — four-way comparison.
3. Why is `ConcurrentHashMap` faster than `Collections.synchronizedMap`? (Per-bucket locking vs global.)
4. What changed in `ConcurrentHashMap` between Java 7 and 8? (Segments → CAS + synchronized on bucket head.)
5. `ArrayList` vs `LinkedList` — when use which? (Almost always `ArrayList`.)
6. What's the `equals`/`hashCode` contract? What breaks if you violate it?
7. Why must keys in a `HashMap` be effectively immutable?
8. `TreeMap` complexity? (O(log n), Red-Black tree.)
9. `LinkedHashMap` — what's special about it? (Insertion or access order; perfect for an LRU cache via `removeEldestEntry`.)
10. `CopyOnWriteArrayList` — when use it? (Many-read, few-write, e.g., listener lists.)

**Evening Block (1 hr) — SYNTHESIS DRILL**

Open your notes. Self-quiz: cover the answer column, ask each question aloud, answer in one breath. Re-read what you fumbled.

**Day 1 success criterion**: You can answer any of the 24 questions above in under 30 seconds with correct technical vocabulary. If a recording of yourself sounds confident, Day 1 worked.

---

### Day 2: Debugging Mechanics + Java Language Fluency

**Theme**: build the dry-run skill and refresh non-concurrency Java basics.

**Morning Block (2.5 hrs) — DEBUGGING DRILLS**

Read Section 9 (Debugging Methodology) end to end. Then practice on real exercises:

The right practice material for this is **harder to find** than LeetCode, because LeetCode is greenfield. Use:

- **Exercism Java track** — has feedback exercises with broken implementations
- **HackerRank "Debugging" section** under Java
- **CodeSignal's "Code Review" exercises**
- **GitHub: search "Java debugging exercises"** — repos like `java-debugging-exercises`
- **Best of all**: take any LeetCode Easy/Medium solution from the discuss tab, deliberately introduce 2–3 bugs (off-by-one, null handling, wrong operator, mutation during iteration), put it down for an hour, then come back and debug it from scratch. Doing this generates infinite practice material and trains the exact skill.

Aim for **6 debugging exercises** today. For each:

1. Read the code top to bottom. State out loud what the class is supposed to do. **Don't predict bugs yet.**
2. Read the failing test case. State expected vs actual output.
3. Dry-run the failing case line by line, voicing variable values at each step.
4. The moment expected diverges from actual, stop. That's the bug zone.
5. Identify the bug type (Section 9 has a taxonomy).
6. State the fix in plain English.
7. Apply the fix.
8. Mentally trace one OTHER test case to make sure your fix doesn't break it.

Don't skip step 8. "Patches that break other tests" is a classic Karat fail.

**Afternoon Block (2 hrs) — JAVA LANGUAGE FLUENCY**

Read Section 8.3 (OOP/Language). Drill aloud:

1. Interface vs abstract class — when each?
2. What changed about interfaces in Java 8? (default methods, static methods.)
3. Method overloading vs overriding — what's resolved at compile time vs runtime?
4. `final` keyword — three contexts (variable, method, class) and what each means.
5. `static` — what does it do? Why is `static` on a non-thread-safe field dangerous?
6. Checked vs unchecked exceptions — give one example of each. When would you create a custom exception?
7. Try-with-resources — what interface does it require? What's the suppressed exception thing?
8. Generics — what is type erasure? Why can't you do `new T()`?
9. PECS — what does it stand for, when does it apply?
10. `Optional` — when is using it appropriate? When is it overkill or wrong? (Answer: don't use for fields, don't use for method params, fine for return types of "may not exist" lookups.)
11. Streams — `map` vs `flatMap`?
12. When are streams a bad choice? (Tight loops, when readability suffers, when you need to short-circuit complex logic.)

**Evening Block (1.5 hrs) — DEBUGGING RETROSPECTIVE + LIGHT PROJECT WORK**

For each of today's 6 debugging exercises, write a one-liner:

> "Bug type: \[off-by-one / null / wrong operator / mutation-during-iter / etc\]. Signal: \[what hinted at it\]. Fix pattern: \[what fixed it\]."

By interview day you want the bug-type taxonomy in muscle memory. When you see code, your eye should jump to "ah, classic off-by-one risk in the loop bound" or "ah, that's mutating while iterating."

**Day 2 success criterion**: You can dry-run a 50-line Java method out loud, calmly tracking 4–5 variables, in under 5 minutes. Speed is less important than smoothness.

---

### Day 3: Extension Practice + Concurrency Code Reading

**Theme**: build the "add a method to existing code" skill and revisit concurrency in *code* form (not just in talk).

**Morning Block (2.5 hrs) — EXTENSION DRILLS**

Read Section 10 (Code Extension Methodology).

Then practice. Source material for this:

- Take any small open-source utility class (Apache Commons has lots — `StringUtils`, `CollectionUtils`)
- Or take a previous LeetCode design problem (`LRUCache`, `MinStack`, `LoggerRateLimiter`) and extend it with a new method
- Or take a piece of your TD code (sanitized — no proprietary logic) and pretend you're being asked to add a method

Aim for **4 extension exercises**. For each:

1. Read the entire class once. Note: naming style (camelCase, prefix conventions), error handling pattern (exceptions vs `Optional` vs null returns), null-check style, helper methods available, mutability of fields.
2. State the contract of the new method aloud — name, parameters, return, side effects, exceptions thrown, edge cases.
3. Identify which existing helpers you'd reuse.
4. Write the method matching the existing style.
5. State two test cases you'd add.

Practice extension articulations on these classes specifically (rebuild from the v1 doc and extend):

- `MinStack` → add `getMax()` in O(1)
- `LRUCache` → add `peek(key)` that doesn't promote the key to most-recent
- `LoggerRateLimiter` → add `setRateLimit(message, seconds)` for per-message limits

Articulate aloud: "I need to add `getMax` to `MinStack` in O(1). The class already uses a parallel min-stack pattern. I'll add a parallel max-stack the same way — push `max(value, currentMax)` on every push, pop both stacks together, return max-stack top on `getMax`. This matches the existing style exactly. Time and space stay O(1) and O(n)."

**Afternoon Block (2 hrs) — CONCURRENCY IN CODE**

This block is reading, not writing. The fundamentals questions in Day 1 were about *talking* concurrency. Today is about *reading* it — because the debugging problem could easily be a concurrent-code bug.

Read these snippets and identify the bug aloud (each one has at least one):

```java
// Snippet A: Singleton
public class Config {
    private static Config instance;
    public static Config getInstance() {
        if (instance == null) {
            instance = new Config();
        }
        return instance;
    }
}
// Bug: not thread-safe. Fix: synchronized method, or DCL with volatile, or initialize at class load.

// Snippet B: Counter
public class Counter {
    private volatile int count;
    public void increment() { count++; }
    public int get() { return count; }
}
// Bug: ++ is not atomic even with volatile. Fix: AtomicInteger, or synchronized.

// Snippet C: Cache
public class Cache {
    private final Map<String, Data> map = new HashMap<>();
    public Data get(String key) {
        if (!map.containsKey(key)) {
            map.put(key, load(key));
        }
        return map.get(key);
    }
}
// Bug: HashMap is not thread-safe; check-then-act is racy regardless. Fix: ConcurrentHashMap.computeIfAbsent.

// Snippet D: Producer/consumer
public class Queue {
    private final List<Item> items = new ArrayList<>();
    public synchronized void put(Item i) { items.add(i); }
    public Item take() {
        if (items.isEmpty()) return null;
        return items.remove(0);
    }
}
// Bug: take() is not synchronized. Reads aren't safe. Plus the design is busy-wait at best. Use BlockingQueue.

// Snippet E: Wait/notify
public synchronized void waitForReady() {
    if (!ready) wait();
}
// Bug: should be while, not if (spurious wakeups). Plus InterruptedException handling missing.
```

Spot-check yourself: each of these 5 patterns is a classic Karat-flavored concurrency-debug question. **Memorize the bug → fix pairs.**

**Evening Block (1 hr) — VOCAB SWEEP**

Re-read Section 8 once. Cover answers, ask aloud, answer aloud. By the end you should be slightly bored — that's the goal. Boredom = retention.

**Day 3 success criterion**: When shown any of the 5 concurrency snippets above, you can name the bug, the fix, and the technical term for the underlying issue ("check-then-act race," "spurious wakeup," "compound action") inside 30 seconds.

---

### Day 4: Mock + Polish

**Theme**: integration. Don't learn anything new.

**Morning Block (2 hrs) — FULL MOCK**

Set a 60-min timer. Run through:

- **15 min — fundamentals rapid-fire**: have someone (or ChatGPT in voice mode, or just a printout) ask you 12 questions from Section 8. Record yourself answering each one. **Do not pause the timer.**
- **25 min — debugging exercise**: pick one of yesterday's debugging exercises you haven't done, or break a fresh piece of code. Dry-run aloud, fix, verify.
- **20 min — extension**: pick one of yesterday's extension targets. Articulate, code, justify.

Then review the recording. Listen to all 60 minutes. Yes, all of it. Note:

- Where did silence happen? (Silence = panic in the recording.)
- Where did you say "um, like, kind of"?
- Where did you use vague vocabulary instead of jargon?
- Where did you skip the dry-run and guess instead?

Write 5 specific bullets to fix.

**Afternoon Block (1.5 hrs) — TARGETED REPAIR**

Re-do the parts where you fumbled. If concurrency vocabulary was weak, drill Section 8.1 again. If your dry-run was sloppy, do one more debugging exercise focused on slowing the trace down.

**Evening Block (1 hr) — POLISH**

- Re-read your one-liner notes (Day 1, 2, 3).
- Re-read the three Talk Tracks (Section 7) aloud once each.
- Set out interview-day materials: water, paper, pen, charger, quiet room.
- **Stop early.** A rested brain on Day 5 outperforms two extra hours on Day 4.

**Day 4 success criterion**: You feel slightly under-stimulated, not panicked. That means the material has stuck.

---

## 6. UMPIRE for Debugging

The original UMPIRE was for greenfield problems. For debugging, the framework adapts:

**U — Understand the existing code (90 sec)**

Read the entire file or function before forming any judgment. Voice what it's *supposed* to do. State the public API. State the data structures used. Do **not** start guessing where bugs might be.

**M — Map the failing case (60 sec)**

Read the failing test. State expected vs actual output. State the input that triggered the failure. If multiple tests fail, pick one to focus on first.

**P — Predict where the divergence happens (30 sec)**

Before tracing, name 2–3 lines or expressions you'd be most suspicious of, and why. (Loops, boundary conditions, the check-then-act pair, etc.) This isn't guessing the answer; it's narrowing where to focus.

**I — Inspect via dry-run (5–8 min)**

Trace the failing case line by line. Out loud. Voice every variable's value. The moment expected diverges from actual, stop.

**R — Repair (3–5 min)**

State the fix in plain English first. Apply it. Run the failed test mentally. Then run **at least one other test case mentally** to confirm you didn't break anything else.

**E — Evaluate (60 sec)**

State why the bug existed (root cause, not just patch). Mention the technical term ("classic check-then-act race," "off-by-one in the loop guard," "missing null check on the optional input"). Note whether the same bug pattern might exist elsewhere in the code.

**Total overhead**: ~5 minutes upfront, ~2 minutes after. Out of a 25-minute slot, ~18 minutes for the actual fix work. Plenty.

---

## 7. Three Talk Tracks

### Talk Track 1 — Fundamentals Rapid-Fire

When asked a fundamentals question, the structure is always: **Definition → Mechanism → Implication.**

Example. "What's the difference between `HashMap` and `ConcurrentHashMap`?"

> "`HashMap` is unsynchronized — it's not safe for concurrent modification, and structural changes from one thread can leave it in an inconsistent state, sometimes even an infinite loop in older versions. `ConcurrentHashMap` is thread-safe but uses fine-grained locking — since Java 8 it locks at the bucket level using CAS for new bucket heads and `synchronized` on the head node for collisions. That's much more concurrent than `Hashtable` or `Collections.synchronizedMap`, which serialize all operations on a single lock. So in practice you reach for `ConcurrentHashMap` whenever you have multi-threaded read/write access; you reach for `HashMap` only inside a single thread or behind your own external synchronization."

Notice: definition (one sentence), mechanism (the bucket-level locking detail), implication (when you'd use which). That's a 25-second answer that scores full marks.

If you don't know the deep mechanism, don't fake it. "I know `ConcurrentHashMap` uses fine-grained locking at the bucket level — I know it's much faster than `Hashtable` under contention, though I don't remember the exact CAS semantics in Java 8 off the top of my head." That's an honest senior answer. Karat scores honesty over made-up confidence.

### Talk Track 2 — Debugging

> "Let me read through this first before I start hypothesizing about bugs.
>
> [Read the code aloud, narrating]
>
> Okay, so this class is supposed to \[contract\]. The public API is \[methods\]. It's using a \[data structures\].
>
> Now let me look at the failing test. The input is \[X\], the expected output is \[Y\], the actual output is \[Z\].
>
> Two areas I'm suspicious of: \[boundary condition X\] and \[shared state Y\]. Let me dry-run the failing case to find out which.
>
> Starting at \[entry point\], with \[initial state\]... \[trace step by step, voicing variables\]... ah — at line 12, when we call \[method\], we get \[wrong value\]. That's the bug.
>
> The root cause is \[technical term — off-by-one, null handling, wrong operator, race condition, etc.\]. The fix is \[describe in plain English\]. Let me apply that.
>
> \[Apply fix\]
>
> Now let me trace one other test case to make sure I haven't broken anything... \[brief trace\]... that still passes. And the failing case now produces \[Y\] as expected.
>
> Worth noting: this same bug pattern could exist at \[other method\] — I'd want to check that too, or add a test for it."

The phrase "let me dry-run the failing case" is critical. **Dry-run is the load-bearing skill.** Karat's rubric for debugging-style problems explicitly looks for systematic tracing vs intuitive guessing.

### Talk Track 3 — Extension

> "Let me first read what's already here so I can match the style.
>
> \[Read the class\]
>
> Okay, so this class \[contract\]. It uses \[conventions: naming, error handling, helpers\]. The new method I need to add is \[X\]. Let me state the contract: name `getFoo`, takes a \[Type\] parameter, returns \[Type\], throws \[Exception\] if \[condition\].
>
> Looking at the existing code, I notice \[useful helper or pattern I can reuse\]. I'll match the existing \[null-check / error-handling\] style — they use \[pattern\], so I'll do the same.
>
> The implementation: \[describe in plain English\]. Time complexity will be \[X\] which matches the rest of the class.
>
> \[Code it\]
>
> Two tests I'd add: \[test 1\] for the happy path, \[test 2\] for the edge case where \[X\]. I'd also confirm that \[existing behavior\] still works — my method shouldn't affect it because I'm only \[explanation\]."

The trap here is to skip "let me first read what's there." Don't. The interviewer is partly checking whether you read existing code before writing new code — exactly the seniority signal Citi cares about.

---

## 8. Java Fundamentals — Expanded

This is the centerpiece of the new plan. Read it slowly. Then drill.

### 8.1 Concurrency (Highest Priority)

**`synchronized`**

Acquires the intrinsic lock (also called the *monitor*) on an object. Reentrant — the same thread can acquire the same lock multiple times without deadlocking itself.

- `synchronized` instance method → locks `this`
- `synchronized` static method → locks the `Class` object
- `synchronized (obj) { ... }` block → locks `obj` specifically

Best practice: lock on a private final dedicated lock object, not on `this`, to prevent external code from interfering with your locking.

**`volatile`**

Guarantees **visibility** — writes by one thread are immediately visible to reads by other threads. The JVM is forbidden from caching the value in registers across the volatile read/write boundary. Establishes a happens-before relationship.

Does **not** guarantee **atomicity**. `volatile int x; x++;` is a read-modify-write; another thread can read between the read and the write, causing a lost update.

Use cases: status flags (`volatile boolean stopped`), the second-check field in double-checked locking.

**`AtomicInteger`, `AtomicLong`, `AtomicReference`**

Lock-free atomic operations using CPU-level CAS (compare-and-swap). When you call `incrementAndGet()`, the CPU atomically reads the current value, computes the new value, and writes it back only if the value hasn't changed since the read. If it changed, retry.

Faster than `synchronized` for simple counters under contention, because no thread ever blocks. But CAS retry storms under high contention can become slow — at very high contention, `LongAdder` is faster (it shards the counter across cells).

**Race condition**

Two or more threads access shared state, and the outcome depends on the timing of their interleaving. Categories: read-modify-write race, check-then-act race, observation race.

**Deadlock**

Two or more threads each hold a lock the other needs. Classic example: thread A holds lock 1 and waits for lock 2; thread B holds lock 2 and waits for lock 1. Prevention: always acquire locks in the same global order; use `tryLock` with timeout; minimize lock scope.

**Livelock**

Threads aren't blocked, but they keep yielding to each other and make no progress. Less common than deadlock.

**Starvation**

A thread can't get a fair shot at the lock — typically because higher-priority threads keep grabbing it. `ReentrantLock(true)` enables fair acquisition.

**Java Memory Model + happens-before**

The JMM defines visibility and ordering guarantees between threads. The fundamental rule: if action A *happens-before* action B, then B sees the effects of A.

Things that establish happens-before:

- Program order within a single thread
- A `synchronized` unlock happens-before subsequent locks on the same monitor
- A `volatile` write happens-before subsequent volatile reads of the same field
- `Thread.start()` happens-before any action in the started thread
- Any action in a thread happens-before another thread's `Thread.join()` returning

Without happens-before, there's no guarantee a write in one thread is ever seen by another. This is why simple shared mutable state without synchronization is broken even on x86 (where you might *think* memory is "strongly ordered" — the JMM is stricter than the hardware).

**`synchronized` vs `ReentrantLock`**

`ReentrantLock` is more flexible:

- `tryLock(timeout)` — give up if you can't acquire in time
- Interruptible acquisition (`lockInterruptibly()`)
- Fair vs unfair (`new ReentrantLock(true)`)
- Multiple `Condition` objects (richer than `wait/notify`)
- Lock and unlock can be in different methods (be careful)

Cost: you must `unlock()` in a `finally` block. With `synchronized`, the JVM unlocks on exception automatically.

**`ReadWriteLock`**

Multiple concurrent readers OR one writer. Use when reads massively outnumber writes. `StampedLock` is even better in Java 8+ for read-heavy workloads via optimistic reads, but the API is more error-prone.

**Thread-safe collections**

- `ConcurrentHashMap` — bucket-level locking. Use for shared mutable maps.
- `ConcurrentSkipListMap` — concurrent sorted map.
- `CopyOnWriteArrayList` — every write copies the entire array. Reads are lock-free. Use for many-read, few-write scenarios (listener lists).
- `BlockingQueue` interface — `ArrayBlockingQueue`, `LinkedBlockingQueue`, `SynchronousQueue`. Producer-consumer patterns.

**`HashMap` vs `Hashtable` vs `Collections.synchronizedMap` vs `ConcurrentHashMap` (the four-way comparison)**

- `HashMap`: unsynchronized. Fast in single-thread.
- `Hashtable`: legacy (Java 1.0). Every method `synchronized` on `this`. Don't use it.
- `Collections.synchronizedMap(new HashMap<>())`: wraps with `synchronized` blocks on a single mutex. Compound operations still need external sync. All operations serialized.
- `ConcurrentHashMap`: per-bucket locking, `computeIfAbsent` / `compute` / `merge` for atomic compound operations. Iterators are weakly consistent (don't throw `ConcurrentModificationException`). The right choice for shared mutable maps.

**`ExecutorService`**

Manages a thread pool. Common factory methods:

- `Executors.newFixedThreadPool(n)` — fixed pool size
- `Executors.newCachedThreadPool()` — unbounded, threads expire after 60s idle
- `Executors.newSingleThreadExecutor()` — serial execution
- `Executors.newScheduledThreadPool(n)` — for delayed/periodic tasks

Submit `Runnable` (no return) or `Callable<T>` (returns `T`). `submit()` returns a `Future<T>`.

`shutdown()` — graceful, lets queued tasks finish.
`shutdownNow()` — interrupts running tasks, returns queued ones.

In production, you usually want a `ThreadPoolExecutor` configured directly so you control queue type, rejection policy, and thread factory.

**`Future` vs `CompletableFuture`**

`Future` is poll-based: `future.get()` blocks. Limited composition.

`CompletableFuture` is callback/composable:

```java
CompletableFuture
    .supplyAsync(() -> fetchUser(id))
    .thenApply(user -> user.getEmail())
    .thenCompose(email -> sendEmail(email))
    .exceptionally(ex -> { log.error(ex); return null; });
```

`thenApply` — sync transformation
`thenCompose` — flat-map (when next step also returns a `CompletableFuture`)
`thenCombine` — combine two futures
`allOf` / `anyOf` — wait for many

**Spring `@Async`**

Marks a method to execute on a separate thread, returning `CompletableFuture<T>` (or `void` for fire-and-forget). Requires `@EnableAsync` on a config class. Caveats: only works on calls from outside the bean (proxy-based), and the executor used can be configured via `AsyncConfigurer`.

You used Spring Boot at TD, so this is in your wheelhouse — be ready to talk about it if they ask about async patterns in real applications.

### 8.2 Collections Internals

**`HashMap` internals**

Array of buckets. Default initial capacity 16, default load factor 0.75. When `size > capacity * loadFactor`, the map resizes (doubles capacity, rehashes all entries — this is expensive).

Insertion: hash the key, mod by capacity to find the bucket, walk the chain to check if the key already exists, append or replace.

Collisions form a singly linked list. Since Java 8, when a bucket has ≥8 entries AND total capacity ≥64, it converts to a balanced tree (red-black) — worst-case lookup goes from O(n) to O(log n). Reverts to a list if size drops below 6.

**`ConcurrentHashMap` internals (Java 8+)**

Per-bucket locking using CAS for empty buckets and `synchronized` on the bucket head node for collision chains. No global lock. Much higher concurrency than `Collections.synchronizedMap`.

`computeIfAbsent`, `compute`, `merge` — atomic compound operations. **Use these instead of get-then-put**, which is a check-then-act race.

Iterators are weakly consistent: they don't throw `ConcurrentModificationException` and may or may not reflect modifications during iteration.

**`ArrayList` vs `LinkedList`**

| | `ArrayList` | `LinkedList` |
|---|---|---|
| Backing | Array | Doubly-linked list |
| Random access | O(1) | O(n) |
| Append (amortized) | O(1) | O(1) |
| Middle insert | O(n) | O(1) at known position, but finding the position is O(n) |
| Cache friendliness | Excellent | Poor |
| Practical use | Almost always | Almost never |

**`ArrayDeque`**

Modern stack and queue. Faster than `Stack` (legacy) and `LinkedList` (poor cache locality). Default for both stack and queue use cases.

**`TreeMap` / `TreeSet`**

Red-Black tree (self-balancing BST). All ops O(log n). Sorted iteration. `floorKey`, `ceilingKey`, `subMap` for range queries.

**`LinkedHashMap`**

`HashMap` + doubly-linked list of entries in insertion order (or access order if configured). Override `removeEldestEntry` to get an LRU cache for free.

**`equals` and `hashCode` contract**

If `a.equals(b)`, then `a.hashCode() == b.hashCode()`. The reverse need NOT hold (collisions are allowed).

If you override `equals` without `hashCode`, the object becomes unfindable in `HashMap`/`HashSet`: you put it in, but `get` or `contains` returns false because the bucket lookup uses `hashCode`.

If a key's hash code changes after it's in a map, it's effectively lost — this is why keys should be effectively immutable.

### 8.3 OOP / Language

**Interface vs abstract class**

Interface = contract. Default methods (Java 8+) allow some implementation, but no instance state.

Abstract class = partial implementation + state.

Modern preference: program to interfaces. Use abstract class only when you genuinely need shared state or shared partial implementation.

**Method overloading vs overriding**

Overloading = same name, different parameters, resolved at **compile time** (static dispatch).
Overriding = subclass replaces a superclass method, resolved at **runtime** (dynamic dispatch).

**`final`**

- `final` variable: can't reassign.
- `final` method: can't override.
- `final` class: can't subclass (e.g., `String`, `Integer`).

`final` parameters and locals don't change behavior, but signal intent.

**`static`**

Class-level, not instance-level. Shared across all instances. Be careful with mutable static state — it's a common source of thread-safety bugs (and test pollution).

**Checked vs unchecked exceptions**

- Checked: must be declared (`throws`) or caught. `IOException`, `SQLException`. Use for recoverable conditions where caller must handle.
- Unchecked: extends `RuntimeException`. `NullPointerException`, `IllegalArgumentException`. Use for programming errors.

Modern bias: lean on unchecked. Checked exceptions don't compose well with streams or lambdas.

**Try-with-resources**

Auto-closes anything implementing `AutoCloseable`. Replaces verbose try/finally. Suppressed exceptions are attached if the close itself throws.

**Generics + type erasure**

Generics are compile-time only. At runtime, `List<String>` is just `List`. You can't do `new T()` because `T` doesn't exist at runtime.

**PECS — Producer Extends, Consumer Super**

- `? extends T` — read-only ("producer"). You read T-or-subtypes out.
- `? super T` — write-only ("consumer"). You write T-or-subtypes in.

`Collection<? extends Number>` lets you read `Number` but not write anything in (compiler can't know if it's `List<Integer>` or `List<Double>`).

**`Optional`**

Use as a return type when "might not exist" is part of the API contract. Don't use for fields. Don't use for method parameters. Don't use `.get()` without `.isPresent()` check (defeats the purpose) — prefer `.orElse`, `.orElseGet`, `.map`, `.ifPresent`.

### 8.4 JVM (Lighter Touch)

- Heap stores objects. Stack stores method frames and local primitives. Each thread has its own stack; heap is shared.
- GC uses the generational hypothesis: most objects die young. Young gen (Eden + Survivors) collected fast and often. Old gen collected less often but more expensive.
- Modern collectors: G1 (default since Java 9), ZGC, Shenandoah (low-latency, large heaps).
- Stop-the-world pauses still happen but are much shorter than older collectors.

You probably won't get deep JVM tuning questions at a screening, so don't spend too much time here.

### 8.5 OOP / Design Principles

SOLID:
- Single Responsibility — one reason to change
- Open/Closed — open for extension, closed for modification
- Liskov Substitution — subtypes substitutable for base type
- Interface Segregation — many small interfaces > one big one
- Dependency Inversion — depend on abstractions, not concretions

Composition vs inheritance: modern preference for composition. Inheritance creates rigid hierarchies; composition lets you mix behavior dynamically.

---

## 9. Debugging Methodology

### Bug Type Taxonomy

When you see a bug in Karat, name the *type* of bug. This signals you've seen it before. The common categories:

**Off-by-one**
- Loop bounds: `<` vs `<=`, `i` vs `i+1`
- Array indexing at boundaries
- Substring methods (Java's `substring(start, end)` is end-exclusive)

**Null handling**
- `NullPointerException` from a method return that wasn't checked
- `null` in a collection causing surprises
- Boxed primitives: `Integer x = null; int y = x;` throws NPE

**Wrong operator**
- `==` vs `equals()` for objects (especially `String`)
- `&` vs `&&`, `|` vs `||`
- `>` vs `>=`
- Integer division when float was intended (`5/2 == 2`, not `2.5`)

**Mutation during iteration**
- `for (X x : list) { list.remove(x); }` → `ConcurrentModificationException`
- Use `Iterator.remove()` or collect-then-remove, or `removeIf`

**Integer overflow**
- `int * int` overflows silently into a negative number
- For binary search: `int mid = (low + high) / 2;` can overflow → use `low + (high - low) / 2`

**Resource leak**
- Streams, connections, file handles not closed
- Use try-with-resources

**Wrong base case**
- Recursion that returns wrong value at base
- Empty input not handled

**Compound action / check-then-act**
- `if (!map.containsKey(k)) map.put(k, v);` is racy in concurrent code; use `putIfAbsent` or `computeIfAbsent`

**Wrong default**
- Uninitialized field defaults: `int = 0`, `boolean = false`, object = `null`
- Sometimes the bug is "should have been initialized to -1, not 0"

**Floating point comparison**
- `0.1 + 0.2 != 0.3` due to floating point representation
- Use a tolerance: `Math.abs(a - b) < epsilon`

**Concurrency**
- Race condition (lost updates, observation race)
- Visibility (one thread's writes never seen by another due to caching) — fix with `volatile` or proper sync
- Deadlock (two locks acquired in inconsistent order)
- `wait` without `while` (spurious wakeup)
- `HashMap` shared without synchronization

### The Dry-Run Discipline

The dry-run is the single most important debugging skill. Practice this until it's smooth:

1. **Open a notepad** (literal paper or a separate window). Write column headers for every variable that matters.
2. **Read each line aloud.** Voice what it does in plain English.
3. **Update the columns** as variables change. Don't trust your head — write it down.
4. **At branches, voice the condition** and which branch is taken.
5. **At loops, count iterations explicitly** — at least the first 2 and the last 2.
6. **At method calls,** voice what's passed in and what's expected to come back.

This sounds slow but it's actually fast in practice — once you have the columns written down, you'll spot the bug almost as soon as it happens.

### What NOT to Do

- ❌ Don't stare and try to spot the bug visually. Trace.
- ❌ Don't guess "I bet it's the loop" without confirming.
- ❌ Don't fix and move on without retracing the failing case to confirm the fix actually fixes it.
- ❌ Don't fix without checking that another test still passes.
- ❌ Don't go silent for 30+ seconds. Voice the trace.

---

## 10. Code Extension Methodology

### Read Before Writing (Always)

Before touching the code, read it once and articulate aloud:

- What does this class do? (One sentence.)
- What are its public methods?
- What internal state does it hold?
- What's the naming style? (`getFoo`, `fetchFoo`, `_foo`?)
- How does it handle errors? Exceptions? Null returns? `Optional`?
- How does it handle null inputs?
- Are there helper methods I can reuse?

This 60-second read-through is almost always skipped by candidates who jump to coding. **Don't be that candidate.** Karat's rubric explicitly looks for "did the candidate read and understand the existing code before extending it."

### State the Contract Before Coding

Before writing the new method:

- **Signature**: name, parameters, return type
- **Preconditions**: what does the input have to look like?
- **Postconditions**: what does the return value mean? What state changes?
- **Edge cases**: empty, null, single-element, duplicate, boundary
- **Exceptions**: what does it throw, and when?
- **Complexity**: time and space, and how it compares to existing methods

Doing this aloud takes 60–90 seconds. It catches design issues before you've written a single line.

### Match the Existing Style

If existing code uses `getOrDefault`, your new code should too — don't suddenly switch to manual `containsKey` checks. If existing code throws `IllegalArgumentException` on bad input, your new method should too — don't suddenly start returning `null`. Style consistency signals seniority.

### Mention Tests

Even if they don't ask for tests, mention them. "I'd add a test for the happy path with input \[X\], plus an edge case test for empty input, plus a test verifying that adding this method doesn't break \[existing behavior\]."

That sentence alone moves you up a tier in the rubric.

### Three Extensions to Drill Specifically

Because they double as "this is a real classic Karat extension":

1. **`MinStack` → add `getMax()` in O(1)**
    - Mirror the min-stack pattern. Push `max(value, currentMax)` on every push.

2. **`LRUCache` → add `peek(key)` that doesn't promote**
    - Look up via the map, return value, but skip the move-to-head step.

3. **`LoggerRateLimiter` → add `setRateLimit(message, seconds)` for per-message custom limits**
    - Add a second map `Map<String, Integer> limits`. Look up the limit; default to global if not set.

For each, articulate the change in under 60 seconds before coding.

---

## 11. DSA Insurance (Light Touch)

Even though the format is debug-and-extend, keep a small DSA capability in case the interviewer deviates or the second problem turns out to be greenfield.

**The minimum viable DSA toolkit (~2 hours total, spread thin across the 4 days):**

- **Two Sum (#1)** — HashMap, one-pass. Re-solve once.
- **Valid Parentheses (#20)** — Stack. Your known gap; re-solve once.
- **Reverse Linked List (#206)** — iterative. Re-solve once.
- **Maximum Depth of Binary Tree (#104)** — recursion. Re-solve once.
- **Number of Islands (#200)** — grid DFS. Optional, do if time permits on Day 3.

If they hand you a greenfield problem and it's not one of these, fall back to: state the brute force, articulate the pattern (Section 7 of the v1 doc), state the optimization, code it. Even if the code isn't perfect, the talk track scores.

**Don't over-invest here.** It's insurance, not the strategy.

---

## 12. Day-of-Interview Playbook

### Night Before
- Light review only: re-read your one-liner notes, the three Talk Tracks, and Section 8.1 (concurrency).
- Test the Karat platform setup if they sent a practice link.
- Lay out water, paper, pen, charger.
- Phone on silent, in another room.
- Sleep early. Rested brain > extra prep.

### 30 Minutes Before
- Re-read your one-liner notes once.
- Re-read Talk Track 2 (debugging) aloud once.
- Stand up. Walk around. Drink water.
- **Don't review code.** At this point, more reading hurts.

### During the Interview

**For fundamentals questions:**
- 1–3 sentences. Definition → Mechanism → Implication.
- Use jargon: "monitor lock," "happens-before," "CAS," "fine-grained locking," "treeification," "compound action."
- If unsure: "I know the high-level — \[X\] — but I don't remember the exact \[Y\] off the top of my head." Honest, not faked.

**For debugging:**
- Read the code first. Voice what it does. Don't guess bugs yet.
- Then read the failing test. State expected vs actual.
- Then dry-run. Out loud. Slow.
- When you find the bug, name the bug type. Apply the fix. Verify with another test mentally.

**For extension:**
- Read the existing code. Voice the conventions.
- State the new method's contract before coding.
- Match the style.
- Mention tests at the end.

**If you get stuck mid-anything:**
- Don't go silent. Verbalize: "Let me think about whether this needs synchronization or whether `volatile` alone is enough. The difference is..."
- Articulated uncertainty reads as engineering. Silence reads as panic.

**Final 2 minutes — questions for them:**
- "What does success look like for someone in this role at 6 months?"
- "What's the team's tech stack and on-call rotation like?"
- "What's the most interesting technical challenge the team is working on right now?"
- Avoid: salary, benefits, "do I need to know X?" — those are for the recruiter.

### Immediately After

- Write down everything you can remember about the questions while it's fresh. Useful for next round and for any future Karat interviews.
- Don't replay it endlessly. The decision is out of your hands now.

---

## 13. What's Cut Now

Time discipline. These are deliberately not in the new plan.

| Topic | Why cut |
|---|---|
| Heavy LeetCode grinding | Format is debug + extend, not greenfield DSA. Insurance only. |
| Tree problems beyond Max Depth | Lower probability now. If asked, fall back on recursion fundamentals. |
| Heap / PriorityQueue problems | Lower probability now. Vocabulary is enough. |
| Graph / Union-Find / Trie | Rare at screening level. |
| Dynamic programming | Hardest pattern, lowest ROI. |
| System design | Not at Karat. Critical for onsite, separate prep. |
| Spring/Spring Boot deep dive | Probably not at Karat screening. You already know it from TD. If they ask about `@Async` or `@Transactional`, you've used both — you can talk about them from experience. Don't drill them. |
| 5-hour YouTube DSA course | Wasted time. |
| Cracking the Coding Interview cover-to-cover | Reference, not prep. |

If the interviewer asks something outside this prep: stay calm, articulate what you'd do from first principles, ask clarifying questions, and try the brute force. A thoughtful struggle beats a panicked surrender.

---

## TL;DR

- **Day 1**: Concurrency fluency + collections internals. Verbal drilling.
- **Day 2**: Debugging mechanics + Java language fluency. Real dry-run practice.
- **Day 3**: Extension drills + concurrency code reading.
- **Day 4**: Mock interview, repair, polish. Stop early.

**Non-negotiables**: No Copilot. Talk out loud. Use jargon. UMPIRE every problem (adapted for debugging). Write the one-liner after every drill.

**Highest-leverage habit**: For every fundamentals answer, structure it as **Definition → Mechanism → Implication** in 25 seconds with correct jargon. This single discipline closes most of the gap between you and a candidate with two more years of prep.

You're not aiming for perfection. You're aiming for: read existing code calmly, dry-run aloud systematically, name bug types with the right jargon, extend code in matching style, articulate tradeoffs out loud. That candidate passes a Citi/Karat screening at 8 YOE. That candidate is reachable in 4 days.

Go.