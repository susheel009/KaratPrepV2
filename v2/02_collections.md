# 02 — Collections Internals

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## 📚 Study Material

### 1. Collection Hierarchy — The Big Picture

```
                        Iterable<T>
                            │
                       Collection<T>
                      ╱      │       ╲
                 List<T>   Set<T>   Queue<T>
                  │         │         │
              ArrayList   HashSet   PriorityQueue
              LinkedList  TreeSet   ArrayDeque
              Vector(L)   LinkedHS  LinkedBlockingQueue
                          EnumSet

                       Map<K,V>  (separate hierarchy — NOT a Collection)
                      ╱    │    ╲
               HashMap  TreeMap  LinkedHashMap
               Hashtable(L)      ConcurrentHashMap
               EnumMap           IdentityHashMap
```

💡 `Map` is **not** a `Collection` — it doesn't implement `Collection<T>`. But `map.entrySet()`, `map.keySet()`, `map.values()` return collections.

### 2. HashMap Internals — Step by Step

**Data structure:** Array of `Node<K,V>` buckets. Each node holds `hash, key, value, next`.

```java
// What happens when you call map.put("Alice", 100):

// Step 1: Compute hash (spread function reduces collisions)
int hash = key.hashCode();             // e.g., "Alice".hashCode() → 63650225
hash = hash ^ (hash >>> 16);           // mix high bits into low bits

// Step 2: Find bucket index
int index = hash & (capacity - 1);     // bitwise AND — equivalent to hash % capacity
                                       // works because capacity is always power of 2

// Step 3: Insert into bucket
// Case A: Bucket empty → create new Node, place directly
// Case B: Bucket has nodes → walk linked list
//   - If key matches (hash equal AND equals() returns true) → replace value
//   - If no match → append to end of list
// Case C: Bucket has ≥ 8 nodes AND capacity ≥ 64 → convert list to RED-BLACK TREE
//   - Tree lookup: O(log n) vs linked list O(n)
//   - Reverts to list when bucket drops to ≤ 6 nodes (hysteresis prevents thrashing)

// Step 4: Check load factor
// If size > capacity × loadFactor (default 0.75) → RESIZE
//   - Double capacity
//   - Rehash every entry (each bucket splits into 2: old index or old index + old capacity)
//   - O(n) — expensive, but amortised over many puts
```

**Key constants:**

| Constant | Value | Why |
|----------|-------|-----|
| Default capacity | 16 | Power of 2 for fast modulo |
| Load factor | 0.75 | Balance between space and collision probability |
| Treeify threshold | 8 | Switch to tree when bucket is this long |
| Untreeify threshold | 6 | Switch back to list (hysteresis gap prevents thrashing) |
| Min treeify capacity | 64 | Don't treeify if table is small — resize instead |

⚠️ **Mutable keys are dangerous:** If you mutate a key after insertion, its `hashCode()` changes, but the entry stays in the old bucket. `get()` computes the new hash → looks in the wrong bucket → returns `null`. The entry is **permanently lost**.

### 3. equals/hashCode Contract

```java
// THE CONTRACT (violate this and HashMap breaks):
// 1. If a.equals(b) → a.hashCode() == b.hashCode()    MUST HOLD
// 2. If a.hashCode() == b.hashCode() → a.equals(b)    NOT REQUIRED (collisions OK)
// 3. hashCode() must return the same value for the same object state

// WHAT BREAKS if you override equals() but NOT hashCode():
class Employee {
    String id;
    @Override public boolean equals(Object o) {
        return o instanceof Employee e && this.id.equals(e.id);
    }
    // hashCode NOT overridden — uses Object.hashCode() (memory address)
}

Employee e1 = new Employee("E001");
Employee e2 = new Employee("E001");
e1.equals(e2);                    // true — same id
map.put(e1, "Alice");
map.get(e2);                     // NULL! — e2 has different hashCode → different bucket
                                  // HashMap never even calls equals() because bucket is wrong
```

**Good hashCode implementation:**
```java
@Override
public int hashCode() {
    return Objects.hash(id, name, department);   // combines all significant fields
}
// Or for records: auto-generated from all components — correct by default
```

### 4. ConcurrentHashMap — Java 7 vs Java 8

```
Java 7: Segment-based locking
┌────────┬────────┬────────┬────────┐
│ Seg 0  │ Seg 1  │ Seg 2  │ Seg 15 │   16 fixed segments
│ lock   │ lock   │ lock   │ lock   │   each segment = mini-HashMap
│ bucket │ bucket │ bucket │ bucket │   threads on different segments don't contend
└────────┴────────┴────────┴────────┘

Java 8: Per-bucket locking (CAS + synchronized on head node)
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   each bucket locks independently
└───┴───┴───┴───┴───┴───┴───┴───┘
  ↑ CAS for empty bucket (lock-free insert)
  ↑ synchronized on head node for non-empty bucket
```

**Why Java 8 is better:**
- Finer granularity — lock per bucket, not per segment
- CAS for empty buckets — no lock at all
- Less memory overhead (no Segment objects)
- Tree bins for long chains (same as HashMap)
- Atomic compound operations: `computeIfAbsent`, `compute`, `merge`

```java
// WRONG — race condition (check-then-act)
if (!map.containsKey(key)) {        // Thread A checks: absent
    map.put(key, compute(key));     // Thread B also checks: absent → DUPLICATE work
}

// RIGHT — atomic compound operation
map.computeIfAbsent(key, k -> compute(k));  // check + insert is ONE atomic operation
map.compute(key, (k, v) -> v == null ? 1 : v + 1);  // atomic read-modify-write
map.merge(key, 1, Integer::sum);   // atomic merge — increment counter
```

### 5. ArrayList vs LinkedList — Why ArrayList Almost Always Wins

```
ArrayList (contiguous array):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ A │ B │ C │ D │ E │ F │ G │ H │  contiguous memory → CPU cache-friendly
└───┴───┴───┴───┴───┴───┴───┴───┘  ↑ index access: base + (index × size) = O(1)

LinkedList (scattered nodes):
┌───┐    ┌───┐    ┌───┐    ┌───┐
│ A │───→│ B │───→│ C │───→│ D │  scattered memory → cache misses
└───┘    └───┘    └───┘    └───┘  each node: 24 bytes overhead (prev + next + header)
```

| Operation | ArrayList | LinkedList |
|-----------|:---------:|:----------:|
| `get(i)` | **O(1)** | O(n) — walk from head/tail |
| `add(e)` (at end) | **O(1) amortised** | O(1) |
| `add(i, e)` (middle) | O(n) — shift elements | O(n) — find position + O(1) insert |
| `remove(i)` | O(n) — shift elements | O(n) — find position + O(1) unlink |
| Memory per element | **~4 bytes** (reference) | ~24 bytes (node overhead) |
| Cache locality | **Excellent** | Poor (random memory access) |

💡 **In practice, ArrayList is faster even for middle insertions** because `System.arraycopy()` is a highly optimised native memory operation, and cache locality dominates performance.

### 6. TreeMap — Sorted Map

Backed by a **red-black tree** (self-balancing BST). All operations O(log n).

```java
TreeMap<String, Integer> tm = new TreeMap<>();
tm.put("Charlie", 3);
tm.put("Alice", 1);
tm.put("Bob", 2);

tm.firstKey();                          // "Alice" — smallest
tm.lastKey();                           // "Charlie" — largest
tm.floorKey("Bob");                     // "Bob" — greatest key ≤ "Bob"
tm.ceilingKey("Brad");                  // "Charlie" — smallest key ≥ "Brad"
tm.subMap("Alice", true, "Charlie", false);  // [Alice, Bob] — range view
tm.headMap("Charlie");                  // {Alice=1, Bob=2} — all before Charlie

// Use when: sorted iteration, range queries, floor/ceiling lookups
// Don't use when: you just need O(1) get/put — use HashMap
```

### 7. LinkedHashMap — Insertion Order + LRU Cache

```java
// Maintains a doubly-linked list threading through all entries
// Iteration order = insertion order (default) or access order

// LRU Cache in 5 lines:
Map<String, Object> lru = new LinkedHashMap<>(16, 0.75f, true) {
    //                         initial capacity ──┘    │     │
    //                         load factor ────────────┘     │
    //                         access order (not insertion) ─┘
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
        return size() > 100;   // evict oldest when cache exceeds 100 entries
    }
};
// On every get() or put(), accessed entry moves to tail of linked list
// Eldest entry (least recently used) is always at head → evicted first
```

### 8. Queue & Deque Implementations

```java
// ArrayDeque — default choice for stack AND queue
Deque<String> stack = new ArrayDeque<>();
stack.push("A");                      // addFirst (stack: LIFO)
stack.pop();                          // removeFirst

Deque<String> queue = new ArrayDeque<>();
queue.offer("A");                     // addLast (queue: FIFO)
queue.poll();                         // removeFirst

// PriorityQueue — min-heap (smallest element polled first)
PriorityQueue<Integer> pq = new PriorityQueue<>();        // natural order (min-heap)
PriorityQueue<Integer> maxPq = new PriorityQueue<>(Comparator.reverseOrder()); // max-heap
pq.offer(5); pq.offer(1); pq.offer(3);
pq.poll();  // 1 — always returns the smallest
// O(log n) insert, O(log n) remove-min, O(1) peek
// NOT sorted — only the head is guaranteed to be the min

// For concurrent: PriorityBlockingQueue, ConcurrentLinkedQueue
```

### 9. Set Implementations

| Implementation | Backed by | Order | Performance | When to use |
|---------------|-----------|-------|-------------|-------------|
| `HashSet` | `HashMap` | None | O(1) add/contains | Default choice |
| `LinkedHashSet` | `LinkedHashMap` | Insertion order | O(1) + linked list overhead | When iteration order matters |
| `TreeSet` | `TreeMap` | Sorted | O(log n) | Range operations, sorted iteration |
| `EnumSet` | Bit vector | Declaration order | O(1) — fastest | Always for enum values |

```java
// EnumSet is the most efficient Set for enum types
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }
Set<Day> weekend = EnumSet.of(Day.SAT, Day.SUN);        // backed by a single long (bit mask)
Set<Day> weekdays = EnumSet.complementOf(weekend);       // all except weekend
Set<Day> all = EnumSet.allOf(Day.class);                 // all values
```

### 10. Immutable Collections (Java 9+)

```java
// Java 9+ factory methods — truly immutable, reject nulls
List<String> list = List.of("a", "b", "c");            // UnsupportedOperationException on add
Set<String>  set  = Set.of("a", "b", "c");             // rejects duplicates (throws)
Map<String, Integer> map = Map.of("a", 1, "b", 2);     // up to 10 entries
Map<String, Integer> bigMap = Map.ofEntries(             // for > 10 entries
    Map.entry("a", 1), Map.entry("b", 2)
);

// Java 10: List.copyOf — immutable defensive copy
List<String> copy = List.copyOf(mutableList);           // independent immutable snapshot

// TRAP: Collections.unmodifiableList(list) is only a VIEW
// Mutations to the original list are visible through the unmodifiable view!
List<String> original = new ArrayList<>(List.of("a", "b"));
List<String> view = Collections.unmodifiableList(original);
original.add("c");          // modifies original
view.size();                // 3! — the view reflects the change
```

### 11. Iterator — Fail-Fast vs Fail-Safe

```java
// FAIL-FAST (ArrayList, HashMap): throws ConcurrentModificationException
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    list.remove(s);      // 💥 ConcurrentModificationException
}
// Fix: use Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();  // safe — modifies through iterator
}
// Or: list.removeIf(s -> s.equals("b"));   // Java 8+ — cleanest

// FAIL-SAFE (CopyOnWriteArrayList, ConcurrentHashMap): works on a snapshot
// No exception, but you might not see concurrent modifications
ConcurrentHashMap<String, Integer> cmap = new ConcurrentHashMap<>();
for (Map.Entry<String, Integer> e : cmap.entrySet()) {
    cmap.remove(e.getKey());  // OK — no CME, weakly consistent iterator
}
```

---

## Rapid-Fire Q&A

### Q1: How does `HashMap` work internally?
**A:** Array of buckets (default capacity 16, load factor 0.75). Hash the key → spread hash (`hashCode ^ (hashCode >>> 16)`) → bucket index = `hash & (capacity - 1)`. Collisions form a linked list. When a bucket has ≥8 entries AND capacity ≥64, it converts to a red-black tree (O(log n) instead of O(n)). Reverts to list at ≤6 entries. Resizes (doubles capacity, rehashes all entries) when `size > capacity × loadFactor`.

### Q2: `HashMap` vs `Hashtable` vs `Collections.synchronizedMap` vs `ConcurrentHashMap`?
**A:**
| | Thread-safe | Locking | null key/value | Use |
|--|:-:|----------|:-:|-----|
| `HashMap` | ❌ | None | ✅/✅ | Single thread |
| `Hashtable` | ✅ | Global lock on `this` | ❌/❌ | Legacy — don't use |
| `synchronizedMap` | ✅ | Global mutex | ✅/✅ | Quick retrofit |
| `ConcurrentHashMap` | ✅ | Per-bucket (CAS + sync on head) | ❌/❌ | Production concurrent |

`ConcurrentHashMap` also provides atomic compound ops: `computeIfAbsent`, `compute`, `merge`.

### Q3: Why is `ConcurrentHashMap` faster than `synchronizedMap`?
**A:** `synchronizedMap` holds a single global lock — all ops serialised. `ConcurrentHashMap` uses per-bucket locking: CAS for empty buckets, `synchronized` on bucket head node for collisions. Multiple threads can read/write different buckets concurrently.

### Q4: What changed in `ConcurrentHashMap` between Java 7 and 8?
**A:** Java 7: segment-based locking (16 fixed segments). Java 8: eliminated segments — uses CAS + `synchronized` on individual bucket head nodes. Better concurrency granularity and reduced memory overhead.

### Q5: `ArrayList` vs `LinkedList` — when use which?
**A:** `ArrayList` almost always. O(1) random access, contiguous memory (cache-friendly). `LinkedList` has O(n) random access, poor cache locality, extra memory per node (prev/next pointers). `LinkedList` is only better for frequent insertions/removals at known positions via iterator — which is rare.

### Q6: What's the `equals`/`hashCode` contract?
**A:** If `a.equals(b)` → `a.hashCode() == b.hashCode()` (must hold). Reverse is NOT required (collisions are allowed). If you override `equals` without `hashCode`, objects become unfindable in `HashMap`/`HashSet` — they go to different buckets. Keys must be effectively immutable — if hashCode changes after insertion, the entry is lost.

### Q7: Why must keys in a `HashMap` be effectively immutable?
**A:** The bucket is determined by `hashCode()` at insertion time. If you mutate the key later, its hashCode changes, but the entry stays in the old bucket. `get()` computes the new hash, looks in the wrong bucket, returns null. The entry is effectively lost.

### Q8: `TreeMap` — complexity? When use it?
**A:** Red-black tree (self-balancing BST). All ops O(log n). Maintains sorted key order. Use for range queries (`subMap`, `floorKey`, `ceilingKey`) or when you need sorted iteration. `NavigableMap` interface.

### Q9: `LinkedHashMap` — what's special?
**A:** `HashMap` + doubly-linked list maintaining insertion order (or access order if constructed with `accessOrder=true`). Override `removeEldestEntry()` to build an LRU cache. Iteration order is predictable — unlike `HashMap`.

### Q10: `CopyOnWriteArrayList` — when use it?
**A:** Every write copies the entire backing array. Reads are lock-free. Use for many-read, few-write scenarios: listener lists, configuration snapshots. Never for write-heavy workloads — O(n) per write.

### Q11: `ArrayDeque` — when use it?
**A:** Resizable array-based deque. Faster than `Stack` (legacy) and `LinkedList` (poor cache locality) for both stack and queue use cases. Default choice for stack (`push/pop`) and queue (`offer/poll`).

### Q12: `Set` implementations — which and when?
**A:** `HashSet`: O(1) add/contains, backed by `HashMap`. `LinkedHashSet`: maintains insertion order. `TreeSet`: sorted order, O(log n) ops, backed by `TreeMap`. `EnumSet`: bitfield-backed, fastest for enum values.

### Q13: `PriorityQueue` — what is it?
**A:** Min-heap. `poll()` always returns the smallest element. O(log n) insert and remove-min. Not thread-safe — use `PriorityBlockingQueue` for concurrent access. Custom ordering via `Comparator`.

### Q14: What's `Collections.unmodifiableList` vs `List.of` vs `List.copyOf`?
**A:** `unmodifiableList(list)`: unmodifiable *view* — original can still be mutated. `List.of(...)`: truly immutable, rejects nulls. `List.copyOf(list)`: immutable copy, rejects nulls. In Java 10+, prefer `List.copyOf` for defensive copies.

### Q15: How does `HashMap` resize work?
**A:** When `size > capacity × loadFactor`, capacity doubles. Every entry is rehashed — bucket index recalculated for new capacity. Each old bucket splits into at most 2 new buckets (old index or old index + old capacity). This is O(n) — expensive.

### Q16: What's `IdentityHashMap`?
**A:** Uses `==` (reference equality) instead of `equals()` for key comparison. Rare use: serialisation frameworks, interning, when you explicitly want reference semantics.

---

## Key Code Patterns

### ConcurrentHashMap — atomic compound operation
```java
// WRONG — check-then-act race
if (!map.containsKey(key)) map.put(key, compute(key));

// RIGHT — atomic
map.computeIfAbsent(key, k -> compute(k));
```

### LRU Cache with LinkedHashMap
```java
Map<String, Object> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
        return size() > MAX_ENTRIES;
    }
};
```

### Immutable collections (Java 10+)
```java
List<String> immutable = List.of("a", "b", "c");          // truly immutable
Map<String, Integer> map = Map.of("one", 1, "two", 2);    // immutable map
Set<String> set = Set.copyOf(mutableSet);                  // immutable copy
```

---

## Can you answer these cold?

- [ ] Walk through `HashMap.put()` — hashing, bucket index, collision, treeification
- [ ] Four-way map comparison — HashMap/Hashtable/synchronizedMap/ConcurrentHashMap
- [ ] `equals`/`hashCode` contract — what breaks if violated
- [ ] `ArrayList` vs `LinkedList` — cache locality argument
- [ ] `ConcurrentHashMap` locking — Java 7 segments vs Java 8 CAS+sync
- [ ] When to use `TreeMap`, `LinkedHashMap`, `CopyOnWriteArrayList`
- [ ] Build an LRU cache with `LinkedHashMap` in 5 lines

[← Back to Index](./00_INDEX.md)
