# 02 — Collections Internals

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/02_collections_doubts.md) · [← Prev: 01 Concurrency](./01_concurrency.md) · [Next: 03 OOP & Language →](./03_oop_language.md)
>
> **Priority:** 🔴 Critical · **Related topics:** [01 Concurrency (concurrent collections)](./01_concurrency.md) · [03 OOP (equals/hashCode, generics)](./03_oop_language.md) · [06 Java 8+ (Streams)](./06_java8_plus.md) · [16 Debugging (mutation during iteration)](./16_debugging.md)
>
> **Practice:** [Karat ladder problems](./Practice/karat-ladder-problems.md) · [DataStructures notes](./DataStructures/DataStructures.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — why so many collection types
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — HashMap put with a collision and a resize
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — minimal map + the equals/hashCode bug
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 The big picture](#41-the-big-picture)
   - [4.2 HashMap put/get walkthrough](#42-hashmap-putget--what-actually-happens)
   - [4.3 equals/hashCode contract](#43-the-equalshashcode-contract-the-rule-that-keeps-the-library-honest)
   - [4.4 ArrayList vs LinkedList](#44-arraylist-vs-linkedlist--cache-locality-wins)
   - [4.5 Iteration safety](#45-iteration-safety--concurrentmodificationexception)
   - [4.6 TreeMap + sorted queries](#46-treemap--sorted-queries-and-range-views)
   - [4.7 LinkedHashMap + LRU cache](#47-linkedhashmap--insertion-order-and-an-lru-cache-in-5-lines)
   - [4.8 Choosing Set / Queue / Deque](#48-choosing-set--queue--deque)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 HashMap internals](#51-hashmap-internals-spread-treeify-resize)
   - [5.2 ConcurrentHashMap: Java 7 vs 8](#52-concurrenthashmap--java-7-segments-vs-java-8-cas--sync)
   - [5.3 Atomic compound operations](#53-atomic-compound-operations-on-concurrenthashmap)
   - [5.4 Generics + erasure (preview)](#54-generics--type-erasure-preview-of-03)
   - [5.5 Fail-fast vs weakly consistent](#55-fail-fast-vs-weakly-consistent-iterators)
   - [5.6 Immutable / unmodifiable](#56-immutable-vs-unmodifiable-vs-copyof)
   - [5.7 Resize mechanics](#57-resize-mechanics-the-cost-of-growing)
   - [5.8 Special-case maps](#58-special-case-maps-identity-enum-weak)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **How to read this file.** §1–§3 build the picture. §4 is everyday tools. §5 is interview depth. §6–§9 are recall and lookup.
> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Picture a small public library, mid-1950s, before computers. The library has thousands of books. A patron walks up: *"Do you have* The Great Gatsby*?"*

How does the librarian find out?

- **Option A — walk every shelf.** Look at every spine. Eventually you hit it (or finish without finding it). For 10 books, fine. For 10,000, the patron is gone by lunchtime. This is **linear scan**: you'll meet it again as `List.contains` and `for (… : list)`.
- **Option B — number every book and put them in numbered slots.** "Book #4,217 is *Gatsby*." Instant lookup *if you know the number*. But you'd have to remember a number for every book — humans don't work that way. This is an **array indexed by position**: instant if you know the index, useless if you only know the title.
- **Option C — a card catalogue.** A wall of small drawers, one per author letter. Inside each drawer, alphabetised cards: each card has the title, author, and a shelf number. To find *Gatsby*: open the "F" drawer (Fitzgerald), flip to "F-i-t", read the shelf number, walk to the shelf. Almost instant. This is a **hash map**: a function (which drawer?) plus a small list inside each drawer.

The library has more problems than fast lookup, though. Some books should only be on the shelf once, even if two donations arrive (a **set**). Some queues should be served in arrival order (a **queue**). Some books should always be returned in alphabetical order to the patron browsing them (a **sorted map**). Each of those is a different physical organisation, with different trade-offs in **lookup speed**, **insertion speed**, **order**, **uniqueness**, and **memory**.

> Once you understand that the library has too many books to scan and that the card catalogue is just a function from book → drawer → shelf, every Java collection type is just a different physical layout that optimises for a different mix of those trade-offs.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

Let's run through the card catalogue — a `HashMap` with **4 drawers** (capacity 4) — adding three books, with one collision, and one resize.

We'll abuse a tiny "hash function" so we can predict where things go: *the number of letters in the title, mod the number of drawers*.

Action | What happens | State
---|---|---
**1.** Add "Dune" → 17 | letters = 4, 4 mod 4 = drawer **0**. Empty → place card directly. | Drawer 0: `["Dune"→17]`
**2.** Add "It" → 9 | letters = 2, 2 mod 4 = drawer **2**. Empty → place card. | D0: `[Dune]`, D2: `[It]`
**3.** Add "1984" → 42 | letters = 4 (treating digits as chars), 4 mod 4 = drawer **0**. **Already has Dune.** Walk drawer 0's list, look for "1984" — not there → append. | **D0: `[Dune, 1984]`** (a collision)
**4.** Lookup "1984" | drawer = 0 → walk the list, compare each card's title with `"1984".equals(card.title)` → match on second card → return 42. | (no change)
**5.** Add "Gatsby" → 11 | letters = 6, 6 mod 4 = drawer **2**. Already has "It" — append. | D0: `[Dune, 1984]`, D2: `[It, Gatsby]`
**6.** Now we have **4 entries / 4 drawers = load 1.0**. The default `HashMap` resize threshold is 0.75 (cap × 0.75 = 3, we hit 4) — **trigger resize**. Capacity doubles to **8**, every card is rehashed (`letters mod 8` now). | D0: `[Dune]` (4 mod 8 = 0), D2: `[It]`, D4: `[1984]` (4 mod 8 = 4 — wait, "1984" also = 4 letters → also drawer 0… but in the real HashMap, the spread function and the way bits work, entries either stay in their old drawer or move to `old + capacity`. We'll redo this properly with real hashes in §3.)

There's the whole `HashMap` story in 6 steps: hash to a drawer, append on collision, equality check on lookup, resize when too crowded.

What this little walkthrough also reveals — and what bites people in interviews:

1. The drawer is chosen by `hashCode()`. The exact card inside is found by `equals()`. **You need both, and they must agree.**
2. If you **mutate the title of a card after filing it**, its hashCode changes, but the card is still in the old drawer. The library can never find it again.
3. The only way to make collisions cheap to walk is to keep the drawer's list short. That's why HashMap resizes.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
import java.util.*;

public class CollectionsDemo {
    public static void main(String[] args) {
        // The card catalogue — Map<title, shelfNumber>
        Map<String, Integer> shelves = new HashMap<>();
        shelves.put("Dune", 17);
        shelves.put("It",   9);
        shelves.put("1984", 42);

        Integer where = shelves.get("1984");   // O(1) average — drawer + short walk
        System.out.println(where);              // 42

        // Now the bug that bites every junior dev exactly once.
        // Define a "key" object that overrides equals() but FORGETS hashCode().
        record BookKey(String title) {
            @Override public boolean equals(Object o) {
                return o instanceof BookKey b && this.title.equals(b.title);
            }
            // hashCode() is NOT overridden — uses Object's default (memory address).
            // Two BookKey("Dune") objects have DIFFERENT memory addresses
            // → DIFFERENT hashCodes → put in DIFFERENT drawers
            // → equals() is never even consulted because the drawers don't match.
        }
        // (Records auto-generate hashCode by default — to demonstrate the bug
        //  we'd write this as a regular class. We're using the record syntax
        //  for brevity; assume hashCode() override removed.)

        Map<BookKey, Integer> byKey = new HashMap<>();
        byKey.put(new BookKey("Dune"), 17);
        Integer found = byKey.get(new BookKey("Dune"));
        // If hashCode were missing: found == null. Yes, even though the keys are .equals().
    }
}
```

What just happened: the first map worked because `String` already provides correctly-paired `equals` and `hashCode`. The buggy version (if you actually delete the record-generated `hashCode`) puts the entry in one drawer and looks for it in another. The lesson: **whenever you override `equals`, you must override `hashCode`.** Records do it for you. Pojos don't.

---

## 4. Build Up — Practical Patterns

### 4.1 The big picture

Java's collections split into two top-level families:

```
            Iterable<T>                            Map<K,V>     ← NOT a Collection
                │                                  │
           Collection<T>                  ┌────────┼────────┐
          ╱     │     ╲                  HashMap   TreeMap   LinkedHashMap
      List<T> Set<T> Queue<T>            Hashtable           ConcurrentHashMap
        │      │      │                                      EnumMap, IdentityHashMap
   ArrayList HashSet PriorityQueue                           WeakHashMap
   LinkedList TreeSet  ArrayDeque
   Vector(legacy) LinkedHashSet  LinkedBlockingQueue
                  EnumSet
```

Choosing the right family is the first decision:

| You need… | Family | Default impl |
|-----------|--------|--------------|
| Indexed positions, duplicates allowed | `List` | `ArrayList` |
| Uniqueness, fast `contains` | `Set` | `HashSet` |
| FIFO / LIFO / priority order | `Queue` / `Deque` | `ArrayDeque` (or `PriorityQueue`) |
| Lookup by key | `Map` | `HashMap` |
| Any of the above, thread-safe | concurrent variants | `ConcurrentHashMap`, `CopyOnWriteArrayList`, `ConcurrentLinkedQueue` |

> 💡 `Map` is **not** a `Collection`. It does not implement `Iterable<T>` or `Collection<T>`. But `map.entrySet()`, `map.keySet()`, `map.values()` all return collections you *can* iterate.

### 4.2 HashMap put/get — what actually happens

```java
// map.put(key, value):
// 1. int h = key.hashCode();                     // pull the user's hashCode
// 2. h = h ^ (h >>> 16);                          // SPREAD: mix high bits into low bits
//                                                 //   (defends against poor hashCodes that
//                                                 //   only vary in the top half of the int)
// 3. int i = h & (table.length - 1);              // bucket index — bitwise AND because
//                                                 //   table.length is always a power of 2
// 4. If table[i] is empty → install Node(h, key, value, next=null).
// 5. If table[i] is non-empty → walk the list (or tree — see §5.1):
//      - if a node has equal hash AND key.equals(node.key)  → replace its value
//      - else                                                → append
// 6. After insert: if size > capacity * loadFactor (default 0.75) → resize() (§5.7).

// map.get(key) repeats steps 1–3, then walks the bucket comparing
// hash AND equals. Returns the matching value or null.
```

The key takeaways: **the bucket is chosen by hashCode**, **the entry inside the bucket is matched by `equals`**, and **the structure resizes when it gets crowded.**

### 4.3 The equals/hashCode contract (the rule that keeps the library honest)

```java
// THE CONTRACT
// 1. If a.equals(b) → a.hashCode() == b.hashCode()    MUST hold.
// 2. If a.hashCode() == b.hashCode() → a.equals(b)    NOT required (collisions are allowed).
// 3. hashCode() must return the same value for the same object state.
//    (i.e. don't change fields used by hashCode after putting the object in a Map/Set.)
```

The most common failure mode — overriding `equals` without `hashCode`:

```java
class Employee {
    final String id;
    Employee(String id) { this.id = id; }

    @Override public boolean equals(Object o) {
        return o instanceof Employee e && this.id.equals(e.id);
    }
    // hashCode NOT overridden — uses Object.hashCode() (essentially memory address).
}

Map<Employee, String> byEmp = new HashMap<>();
byEmp.put(new Employee("E001"), "Alice");
byEmp.get(new Employee("E001"));   // null — second Employee has a DIFFERENT hashCode,
                                   // looks in a different bucket, equals() never runs.
```

**Fix:** generate both via `Objects.equals` / `Objects.hash`, or use a `record`:

```java
@Override public int hashCode() { return Objects.hash(id); }   // covers all fields used in equals

// Or, simpler:
public record Employee(String id) {}      // auto-generates correct equals + hashCode
```

### 4.4 ArrayList vs LinkedList — cache locality wins

```
ArrayList — one contiguous array of references:
[ A | B | C | D | E | F | G | H ]    indexed access: address + (i × 4 bytes) → O(1)
                                     contiguous → CPU prefetcher loves it

LinkedList — scattered nodes connected by pointers:
[A]→[B]    [C]→[D]      [E]         each node = 24+ bytes (prev + next + header)
       ↘   ↗     ↘    ↗              addresses chosen by the allocator → cache-miss city
        [F]       [G]
```

| Operation | `ArrayList` | `LinkedList` |
|-----------|:-----------:|:------------:|
| `get(i)` | **O(1)** | O(n) — walk from head/tail |
| `add(e)` (end) | **O(1) amortised** | O(1) |
| `add(i, e)` (middle) | O(n) shift | O(n) walk + O(1) insert |
| `remove(i)` (middle) | O(n) shift | O(n) walk + O(1) unlink |
| Memory per element | ~4 bytes | ~24 bytes |
| Cache locality | **Excellent** | Poor |

> 💡 In practice, `ArrayList` even wins on middle insertions because `System.arraycopy` is a tight, cache-friendly native memcpy. **Default to `ArrayList`. Use `LinkedList` only when you have measured evidence that head/tail O(1) `Deque` ops matter — and then strongly consider `ArrayDeque` instead.**

### 4.5 Iteration safety — `ConcurrentModificationException`

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// CRASH — fail-fast iterator detects mutation between iterations
for (String s : list) {
    if (s.equals("b")) list.remove(s);   // 💥 ConcurrentModificationException
}

// FIX 1 — Iterator.remove() — safe because the iterator updates its own state
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();
}

// FIX 2 — removeIf (Java 8+) — cleanest
list.removeIf(s -> s.equals("b"));

// FIX 3 — collect to remove, then removeAll
List<String> toRemove = list.stream().filter(s -> s.startsWith("b")).toList();
list.removeAll(toRemove);
```

`ConcurrentHashMap` and `CopyOnWriteArrayList` give you **weakly-consistent** iterators instead — no exception, but the iterator may or may not reflect concurrent changes (more in §5.5).

### 4.6 TreeMap — sorted queries and range views

```java
TreeMap<String, Integer> tm = new TreeMap<>();
tm.put("Charlie", 3);
tm.put("Alice", 1);
tm.put("Bob", 2);

tm.firstKey();                           // "Alice" — smallest
tm.lastKey();                            // "Charlie" — largest
tm.floorKey("Bob");                      // "Bob" — greatest key ≤ "Bob"
tm.ceilingKey("Brad");                   // "Charlie" — smallest key ≥ "Brad"
tm.subMap("Alice", true, "Charlie", false);  // {Alice=1, Bob=2} — half-open range view
```

Backed by a **red-black tree** (self-balancing BST). All operations O(log n). Use when you need: sorted iteration, range queries, floor/ceiling. Otherwise `HashMap` is faster.

### 4.7 LinkedHashMap — insertion order and an LRU cache in 5 lines

```java
// LinkedHashMap = HashMap + a doubly-linked list threading every entry.
// Iteration order = insertion order by default,
// or "access order" if you pass `accessOrder=true`.

Map<String, Object> lru = new LinkedHashMap<>(16, 0.75f, /*accessOrder*/true) {
    @Override protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
        return size() > 100;     // evict the head (least recently used) when over capacity
    }
};
// On every get() and put(), the touched entry moves to the tail of the linked list.
// The head is always the LRU entry — LinkedHashMap removes it automatically.
```

### 4.8 Choosing Set / Queue / Deque

| You need… | Use | Why |
|-----------|-----|-----|
| Plain unique items | `HashSet` | O(1) backed by `HashMap` |
| Unique items, iteration order | `LinkedHashSet` | insertion-ordered |
| Sorted unique items | `TreeSet` | O(log n), `floor`/`ceiling`/`subSet` |
| Enum-typed set | `EnumSet` | bit-vector, fastest |
| Stack (LIFO) | `ArrayDeque` (`push`/`pop`) | beats legacy `Stack`, beats `LinkedList` on cache |
| FIFO queue | `ArrayDeque` (`offer`/`poll`) | same |
| Priority order | `PriorityQueue` | min-heap, O(log n) |
| Concurrent FIFO | `ConcurrentLinkedQueue` (lock-free) or `LinkedBlockingQueue` (blocking, bounded) | producer-consumer ([§5.7 of 01_concurrency](./01_concurrency.md#57-blockingqueue-and-the-producer-consumer-pattern)) |

---

## 5. Going Deep — Interview-Level Material

### 5.1 HashMap internals (spread, treeify, resize)

**Hash spread.** A user's `hashCode()` may have weak entropy in the low bits (e.g. `Integer.hashCode()` returns the int itself). The bucket index is `hash & (capacity - 1)` — only the low bits matter. So `HashMap` mixes high bits down with `h ^ (h >>> 16)` before masking. This single XOR turns a lot of bad-but-not-malicious hashCodes from O(n) buckets into O(1) buckets.

**Treeification (Java 8+).** When a bucket exceeds **8 entries** *and* table capacity is **≥ 64**, the bucket converts from a singly-linked list to a **red-black tree**. Lookup in that bucket changes from O(n) to O(log n). A bucket reverts to a list when it shrinks to **6 entries** (the 8/6 hysteresis gap prevents thrashing on adversarial workloads). Below capacity 64, an over-stuffed bucket triggers a resize instead of a tree.

| Constant | Value | Reason |
|----------|-------|--------|
| Default capacity | 16 | Power of 2 (fast `& mask`) |
| Load factor | 0.75 | Empirical balance: collision rate vs memory |
| Treeify threshold | 8 | Probability of a bucket reaching this with a good hash is < 10⁻⁶ |
| Untreeify threshold | 6 | Hysteresis gap |
| Min treeify capacity | 64 | Below this, resize first |

> ⚠ **Mutable-key bug.** Mutate any field used in a key's `hashCode()` after insertion → the entry stays in the old bucket → `get()` looks in the new bucket → `null`, forever. Treat keys as immutable.

### 5.2 ConcurrentHashMap — Java 7 segments vs Java 8 CAS + sync

```
Java 7: 16 fixed Segments, each with its own lock
[Seg 0 lock][Seg 1 lock]…[Seg 15 lock]      threads on different segments don't contend
   table       table         table          but two threads on the same segment serialise

Java 8+: lock per BUCKET (no segments)
[Bucket 0 ][Bucket 1 ]…[Bucket N ]
   ↑ empty bucket: insert via CAS, lock-free
   ↑ non-empty bucket: synchronized on the head Node
```

Java 8's per-bucket model gives finer granularity (locks scale with capacity, not a fixed 16), supports tree-bins on long collisions just like `HashMap`, and frees memory previously eaten by segment objects.

> ⚠ **Both versions still reject `null` keys and `null` values** — unlike plain `HashMap`, which allows both. The reason: in a concurrent map, `get` returning `null` would be ambiguous between "key absent" and "key present with `null` value", which can't be resolved without locking.

### 5.3 Atomic compound operations on ConcurrentHashMap

```java
// WRONG — check-then-act race condition. Two threads both see "absent",
// both compute, both put. Duplicate work and last-write wins.
if (!map.containsKey(key)) {
    map.put(key, expensive(key));
}

// RIGHT — atomic compound op. Compute runs at most once per absent key.
map.computeIfAbsent(key, k -> expensive(k));

// Other atomic compound ops:
map.compute(key, (k, v) -> v == null ? 1 : v + 1);   // atomic read-modify-write
map.merge(key, 1, Integer::sum);                       // atomic increment counter
map.putIfAbsent(key, value);                           // atomic "set only if absent"
```

> ⚠ **The lambda you pass to `computeIfAbsent` must be short and non-blocking.** It runs while the bucket is locked — slow lambdas serialise everything else hitting that bucket. **Never call back into the same map** in the lambda — that can deadlock or recurse.

### 5.4 Generics + type erasure (preview of [03 OOP](./03_oop_language.md))

`List<String>` is the same as `List<Integer>` at runtime — `T` is erased. Two consequences that matter for collections specifically:

```java
List<String>  ls = new ArrayList<>();
List<Integer> li = new ArrayList<>();
ls.getClass() == li.getClass();        // true — both ArrayList.class

// Cannot create generic arrays:
T[] arr = new T[10];                   // ❌ won't compile
List<String>[] map = new List<String>[10];   // ❌ won't compile
List<String>[] map = (List<String>[]) new List[10];  // ⚠ unchecked cast — works but warns
```

Heap pollution and `@SafeVarargs` are the two other gotchas — covered in [03 §5](./03_oop_language.md#5-going-deep--interview-level-material).

### 5.5 Fail-fast vs weakly consistent iterators

| Iterator type | Behaviour on concurrent mutation | Examples |
|---------------|----------------------------------|----------|
| **Fail-fast** | Throws `ConcurrentModificationException` if structure changes during iteration | `ArrayList`, `HashMap`, `HashSet`, `TreeMap` |
| **Weakly consistent** | No exception. Reflects state at construction time; may or may not see later changes. | `ConcurrentHashMap`, `ConcurrentSkipListMap`, `CopyOnWriteArrayList` |
| **Snapshot** | Iterates a frozen snapshot at construction | `CopyOnWriteArrayList` (the array reference at iteration start) |

Fail-fast is **best-effort** — it uses an internal `modCount` counter, and the JMM gives no real guarantee across threads. So fail-fast iterators are useful for catching single-threaded bugs (mutation during iteration), but **not** a safety mechanism in concurrent code.

### 5.6 Immutable vs unmodifiable vs `copyOf`

Three things that look similar but aren't:

```java
// 1. UNMODIFIABLE WRAPPER (Java 1.2) — VIEW, not a copy
List<String> backing = new ArrayList<>(List.of("a", "b"));
List<String> view    = Collections.unmodifiableList(backing);
backing.add("c");
view.size();          // 3 — the wrapper sees backing's mutations!

// 2. IMMUTABLE FACTORY (Java 9+) — TRULY immutable, REJECTS nulls
List<String> imm = List.of("a", "b", "c");
imm.add("d");         // UnsupportedOperationException
List.of("a", null);   // NullPointerException — unlike ArrayList which allows nulls

// 3. COPY (Java 10+) — SNAPSHOT immutable, independent of source
List<String> snap = List.copyOf(backing);
backing.add("c");
snap.size();          // 2 — snap has its own array, backing's change invisible
```

> 💡 **Defensive-copy idiom in modern Java:** `List.copyOf(input)` — one line, immutable, null-safe, optimised (returns the input unchanged if it's already immutable).

### 5.7 Resize mechanics — the cost of growing

When `size > capacity × loadFactor` (default 16 × 0.75 = 12), `HashMap` doubles its table and rehashes every entry. Cost: **O(n)**, where n = current size. Single insertions become slow on the resize step but amortise out — total cost of n inserts is O(n).

A subtle Java 8+ optimisation: when capacity doubles from `C` to `2C`, every entry's new bucket index is **either** the old index **or** old index + C. The HashMap source uses this to split each old bucket into two new buckets in a single linked-list walk — no full rehash needed for entries already correctly placed.

> 💡 **Pre-sizing trick:** if you know you're inserting N entries, construct with `new HashMap<>((int) Math.ceil(N / 0.75) + 1)` to avoid every resize during the loading phase.

### 5.8 Special-case maps: identity, enum, weak

```java
// IdentityHashMap — uses == not equals for keys.
//   Use: serialisation graph traversal (cycle detection), object identity tracking.
IdentityHashMap<Object, String> ids = new IdentityHashMap<>();

// EnumMap — array indexed by enum.ordinal(). Tiny, fast, ordered by enum declaration.
//   Use: ANY map keyed by an enum.
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }
EnumMap<Day, Integer> hours = new EnumMap<>(Day.class);

// WeakHashMap — keys held by WeakReference. When key has no strong refs elsewhere,
// GC reclaims the key and its entry vanishes from the map.
//   Use: caches keyed by mutable objects you don't own. Beware: values still strongly held;
//   if values reference keys you've created an unbreakable strong cycle.
WeakHashMap<Object, byte[]> cache = new WeakHashMap<>();
```

---

## 6. Memory Aids

### Decision tree: "I need a collection. Which one?"

```
Need key→value lookup?
└── Yes → Map
    ├── Single thread, no order needed?  → HashMap
    ├── Insertion order or LRU?           → LinkedHashMap (override removeEldestEntry)
    ├── Sorted by key?                    → TreeMap
    ├── Concurrent?                       → ConcurrentHashMap
    ├── Keyed by enum?                    → EnumMap
    └── Identity (==) keys?               → IdentityHashMap

Need uniqueness without lookup-by-key?
└── Yes → Set
    ├── HashSet (default)
    ├── LinkedHashSet (insertion order)
    ├── TreeSet (sorted)
    └── EnumSet (enum values — bit-vector backed, fastest)

Need positional / indexed access?
└── Yes → List
    ├── ArrayList (default — almost always)
    └── LinkedList (rare — only for known head/tail Deque ops; ArrayDeque is usually better)

Need stack / queue / priority?
└── Yes → Queue / Deque
    ├── ArrayDeque (default for both stack and queue)
    ├── PriorityQueue (min-heap)
    ├── ArrayBlockingQueue (bounded, producer-consumer with backpressure)
    ├── LinkedBlockingQueue (optionally bounded, higher throughput)
    └── ConcurrentLinkedQueue (lock-free FIFO)
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Is `Map` a `Collection`?" | NO | "Map is a separate hierarchy. But `entrySet()` / `keySet()` / `values()` are Collections." |
| "What's `HashMap`'s big-O?" | "Average O(1), worst O(log n) since Java 8" | Treeification at 8 entries / capacity ≥ 64. |
| "Why two thresholds 8 and 6?" | Hysteresis | Prevents thrashing back and forth between list and tree. |
| "Why does `equals` need `hashCode`?" | "The bucket is chosen by hashCode" | Without matching hashCode, equal objects sit in different buckets, never compared. |
| "`ArrayList` or `LinkedList`?" | ArrayList | "Cache locality dominates. Use ArrayDeque for stack/queue, not LinkedList." |
| "How does `ConcurrentHashMap` differ from `synchronizedMap`?" | "Per-bucket vs global lock" | Java 8+: CAS for empty buckets, sync on head node for non-empty. Plus atomic computeIfAbsent. |
| "How do I build an LRU cache?" | LinkedHashMap with accessOrder | Override removeEldestEntry. |

### Three anchor problems

1. **Lookup speed depends on the bucket** — hashCode chooses; equals confirms.
2. **Mutation invalidates lookup** — a key whose hashCode changes after insertion is lost.
3. **Concurrent ≠ synchronized** — ConcurrentHashMap is many small locks; synchronizedMap is one big lock.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: How does `HashMap` work internally?
**A:** Array of buckets (default 16, load factor 0.75). Hash the key → spread (`h ^ (h >>> 16)`) → bucket index = `hash & (capacity - 1)`. Collisions form a linked list; ≥ 8 entries with capacity ≥ 64 → red-black tree (Java 8+). Reverts at ≤ 6 entries. Resize doubles capacity and rehashes when `size > capacity × loadFactor`.

### Q2: `HashMap` vs `Hashtable` vs `Collections.synchronizedMap` vs `ConcurrentHashMap`?
**A:**
| | Thread-safe | Locking | null key/value |
|--|:-:|---------|:-:|
| `HashMap` | ❌ | none | ✅/✅ |
| `Hashtable` | ✅ | global on `this` | ❌/❌ |
| `synchronizedMap` | ✅ | global mutex | ✅/✅ |
| `ConcurrentHashMap` | ✅ | CAS + per-bucket sync | ❌/❌ |

`ConcurrentHashMap` also provides atomic `computeIfAbsent`, `compute`, `merge`.

### Q3: Why is `ConcurrentHashMap` faster than `synchronizedMap`?
**A:** `synchronizedMap` holds one global lock — every operation serialises. `ConcurrentHashMap` uses per-bucket locking (CAS for empty buckets, `synchronized` on the head node for collisions). Different buckets are independent; many threads can read/write concurrently.

### Q4: What changed in `ConcurrentHashMap` between Java 7 and 8?
**A:** Java 7: 16 fixed segments, each with its own lock. Java 8: segments removed; locking is per-bucket via CAS + `synchronized` on the bucket head. Better granularity, less memory overhead, supports tree-bins.

### Q5: `ArrayList` vs `LinkedList` — when use which?
**A:** `ArrayList` almost always. O(1) random access, contiguous memory, cache-friendly. `LinkedList` has O(n) random access and 24+ bytes overhead per node. For stack/queue use cases, `ArrayDeque` beats both.

### Q6: What's the `equals`/`hashCode` contract?
**A:** If `a.equals(b)` → `a.hashCode() == b.hashCode()` (must hold). Reverse not required (collisions OK). hashCode must be stable for the object's lifetime in a hash collection. Override both together — never just one.

### Q7: Why must keys in a `HashMap` be effectively immutable?
**A:** Bucket is chosen by `hashCode()` at insertion. Mutating a field that contributes to hashCode changes the hash → entry stays in the old bucket → `get()` searches the new bucket → returns null. Entry is permanently lost.

### Q8: `TreeMap` — complexity? When use it?
**A:** Red-black tree. All ops O(log n). Maintains sorted key order. Use for range queries (`subMap`, `floorKey`, `ceilingKey`) or sorted iteration.

### Q9: `LinkedHashMap` — what's special?
**A:** `HashMap` plus a doubly-linked list threading entries. Iteration order = insertion order, or access order if constructed with `accessOrder=true`. Override `removeEldestEntry` to build an LRU cache.

### Q10: `CopyOnWriteArrayList` — when use it?
**A:** Every mutation copies the entire backing array. Reads are lock-free. Ideal for many-read, few-write scenarios: listener lists, config snapshots. Never for write-heavy workloads — O(n) per write.

### Q11: `ArrayDeque` — when use it?
**A:** Resizable circular array. Default for both stack (`push`/`pop`) and queue (`offer`/`poll`). Beats legacy `Stack` (synchronised) and `LinkedList` (poor cache locality).

### Q12: `Set` implementations — which and when?
**A:** `HashSet` default (O(1)). `LinkedHashSet` for insertion order. `TreeSet` for sorted order (O(log n)). `EnumSet` for enum-typed sets — bit-vector, fastest.

### Q13: `PriorityQueue` — what is it?
**A:** Binary min-heap. `poll()` returns the smallest element. O(log n) insert and remove-min. NOT sorted on iteration (only the head is the minimum). Custom order via `Comparator`. For concurrent: `PriorityBlockingQueue`.

### Q14: `Collections.unmodifiableList` vs `List.of` vs `List.copyOf`?
**A:** `unmodifiableList(list)` — view, original mutations are visible. `List.of(...)` — truly immutable, rejects nulls (Java 9+). `List.copyOf(list)` — immutable snapshot, independent of source (Java 10+). Modern defensive copies use `List.copyOf`.

### Q15: How does `HashMap` resize work?
**A:** When `size > capacity × loadFactor`, capacity doubles. Every entry rehashed. Java 8+ optimisation: each old bucket splits into two new buckets (old index or old index + old capacity). O(n) — amortised.

### Q16: What's `IdentityHashMap`?
**A:** Uses `==` instead of `equals`. Rare uses: serialisation graph cycle detection, debugger / profiler tooling, interning. Never use it for general-purpose lookup.

### Q17: Why are `null` keys/values rejected by `ConcurrentHashMap`?
**A:** In a concurrent map, `get(k)` returning `null` would be ambiguous between "key absent" and "key present, value null" without holding a lock. Forbidding nulls keeps `get` lock-free and unambiguous.

### Q18: Fail-fast vs weakly consistent iterators?
**A:** Fail-fast (`ArrayList`, `HashMap`): throw `ConcurrentModificationException` on structural change during iteration — best-effort, useful for catching single-thread bugs. Weakly consistent (`ConcurrentHashMap`, `CopyOnWriteArrayList`): no exception; reflects state at construction; may or may not see later changes.

### Q19: PECS — what does it mean for collections?
**A:** Producer Extends, Consumer Super. `Collection<? extends T>` lets you read T-or-subtypes (producer). `Collection<? super T>` lets you write T-or-subtypes (consumer). `Collections.copy(dest, src)` is `copy(List<? super T> dest, List<? extends T> src)` — textbook PECS.

### Q20: What are the treeification thresholds and why?
**A:** A bucket converts to a red-black tree at **8** entries (untreeify at **6** — hysteresis gap), provided table capacity ≥ **64**. Probability of reaching 8 with a well-distributed hash is < 10⁻⁶, so it only triggers on bad hashes / adversarial inputs. Below capacity 64, resize first.

### Q21: `computeIfAbsent` vs `containsKey + put`?
**A:** `computeIfAbsent(k, fn)` is atomic — the function runs at most once even under concurrent access (on `ConcurrentHashMap`). `containsKey` followed by `put` is a check-then-act race: two threads both see absent and both put. Always prefer the atomic compound op.

---

### Key Code Patterns

**Atomic compound operation on `ConcurrentHashMap`**
```java
counters.merge(key, 1L, Long::sum);                   // atomic increment
caches.computeIfAbsent(key, k -> expensiveLoad(k));   // compute at most once per key
```

**LRU cache with `LinkedHashMap`**
```java
Map<K, V> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > MAX; }
};
```

**Immutable defensive copy (Java 10+)**
```java
this.tags = List.copyOf(input);   // null-safe, immutable, optimised
```

**Pre-sized HashMap to avoid resizes**
```java
Map<K, V> sized = new HashMap<>((int) Math.ceil(expectedCount / 0.75) + 1);
```

**Safe in-place removal during iteration**
```java
list.removeIf(e -> e.isExpired());                    // Java 8+, cleanest
```

---

## 8. Self-Test

**Easy**
- [ ] What's the default capacity and load factor of `HashMap`?
- [ ] Is `Map` a `Collection`?
- [ ] What does `ArrayDeque.push` do?
- [ ] Why must `equals` and `hashCode` be overridden together?

**Medium**
- [ ] Walk through `HashMap.put("foo", 1)` end to end — spread, bucket index, collision, treeify, resize.
- [ ] What breaks if you mutate a HashMap key after insertion? Why?
- [ ] Compare `ArrayList` and `LinkedList` for: random access, middle insert, memory overhead, cache locality.
- [ ] How would you build an LRU cache with `LinkedHashMap`?
- [ ] What's the difference between `unmodifiableList`, `List.of`, and `List.copyOf`?

**Hard**
- [ ] Compare `ConcurrentHashMap` Java 7 vs Java 8 — locking, structure, atomic ops.
- [ ] Why is `computeIfAbsent` atomic but `if (!map.containsKey(k)) map.put(k, v)` a race?
- [ ] Why does `ConcurrentHashMap` reject null keys and values?
- [ ] Explain the 8/6 hysteresis gap on tree-bin conversion.
- [ ] What's a fail-fast iterator? Why isn't it a safety mechanism in concurrent code?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Bucket** | A slot in the hash table's array. The hash function picks one. |
| **Collision** | Two distinct keys mapping to the same bucket. |
| **Load factor** | Fill ratio at which `HashMap` doubles its table. Default 0.75. |
| **Treeification** | Switching a long collision chain (≥ 8) to a red-black tree (Java 8+). |
| **Hysteresis** | The 8/6 gap that prevents flip-flopping between list and tree. |
| **CME** | `ConcurrentModificationException` — fail-fast iterator complaint about mutation during iteration. |
| **Fail-fast iterator** | Throws CME on structural change. Best-effort. |
| **Weakly consistent iterator** | No exception on concurrent change; reflects state-at-construction with possible later changes. |
| **Heap pollution** | A generic collection holding values whose runtime types don't match the declared type parameter (only possible via raw types or unchecked casts). |
| **PECS** | Producer Extends, Consumer Super — wildcard placement rule. |
| **CAS** | Compare-and-swap; atomic CPU instruction. Backbone of `ConcurrentHashMap`'s lock-free path. |
| **Atomic compound operation** | A check-then-act executed as one indivisible step (`computeIfAbsent`, `merge`, `compute`). |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/02_collections_doubts.md) · [← Prev: 01 Concurrency](./01_concurrency.md) · [Next: 03 OOP & Language →](./03_oop_language.md)

[↑ Back to top](#02--collections-internals)
