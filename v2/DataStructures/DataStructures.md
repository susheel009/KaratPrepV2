# Data Structures in Java: From the Ground Up

A walkthrough of every data structure that matters for interview-level DSA in Java — starting with where the word came from, what it looks like in everyday life, what problem it actually solves, how Java exposes it, the methods you'll actually use, the traps that bite in predict-the-output rounds, and the kinds of LeetCode patterns it shows up in.

---

## Foundation: What "Data Structure" Even Means

**Etymology.** *Data* is Latin, plural of *datum* — literally "a thing given." *Structure* comes from Latin *struere*, "to build, to arrange in layers" (same root as *construct* and *destroy*). A data structure is, word-for-word, **"an arrangement of given things."**

**Why we need different ones.** Memory is just a long row of bytes. Every data structure is a *convention* layered on top of that row to make some operation cheap (lookup, insert, sort, find-min) at the cost of making other operations more expensive. Picking a data structure is picking which trade-offs you want.

### Mental Model Glossary

Before diving in, four ideas underpin every trade-off below:

- **Big-O** — how cost scales as input grows, ignoring constants. `O(1)` = constant, `O(log n)` = halves each step, `O(n)` = scans, `O(n log n)` = best comparison sort, `O(n²)` = nested scans.
- **Amortized cost** — average per-operation cost over a long sequence. `ArrayList.add` is *amortized* O(1) because the occasional resize copy is rare enough to average out.
- **Cache locality** — memory laid out contiguously is much faster to traverse than memory scattered via pointers, because CPU caches load whole adjacent chunks. This is why `ArrayList` usually beats `LinkedList` even though `LinkedList` has "better" theoretical inserts.
- **Reference vs value semantics** — Java collections store references to objects. Mutating the object after putting it in a `HashSet`/`HashMap` key can break the structure. This is a classic predict-the-output trap.

### How to Read Each Section

Every data structure below follows the same shape so you can scan or deep-read:

1. **Etymology** — a hook for memory
2. **Real-world example** — the metaphor
3. **Problem it solves** — when to reach for it
4. **Complexity at a glance** — the table you'll memorize
5. **Methods cheat sheet** — runnable code, every method you'll use
6. **Under the hood** — internals (only where they matter)
7. **Gotchas & predict-the-output traps** — what bites in interviews
8. **Pattern template** — the boilerplate for the most common interview use
9. **Expected DSA problem types** — LeetCode patterns

---

## 1. Array

### Etymology
From Old French *areer / arei*, meaning **"to arrange in order"** — especially troops in battle formation. "An array of soldiers" was a row of soldiers in fixed positions. The meaning leaked into mathematics and then computing.

### Real-world example
An **egg carton**. Twelve numbered slots, fixed at manufacture time. To get egg #7, you don't search — you reach directly to slot 7. The slots have a known size and a known position relative to the start of the carton.

### Problem it solves
"I have N items of the same kind. I want to grab item #i instantly without looking at the others." Without arrays, you'd have to walk through every item one by one.

### Complexity at a glance

| Operation                | Time   | Notes                                       |
|--------------------------|--------|---------------------------------------------|
| Access by index `arr[i]` | O(1)   | Address arithmetic                          |
| Update `arr[i] = x`      | O(1)   |                                             |
| Search (unsorted)        | O(n)   | Linear scan                                 |
| Search (sorted)          | O(log n) | Binary search via `Arrays.binarySearch`   |
| Insert at end            | N/A    | Fixed size — can't insert                   |
| Insert in middle         | O(n)   | Only via copy to new array                  |
| **Space**                | O(n)   | Contiguous block                            |

### Methods cheat sheet
```java
import java.util.Arrays;

// Declare and initialize
int[] a = new int[5];                 // [0,0,0,0,0] — primitives default to 0
int[] b = {3, 1, 4, 1, 5, 9, 2, 6};   // literal init
Integer[] c = new Integer[5];         // [null,null,null,null,null] — boxed defaults to null
int[][] grid = new int[3][4];         // 2D array, all zeros
int[][] jagged = new int[3][];        // rows declared, columns set per row

// Basic access
int len = b.length;                   // FIELD, not method — no parentheses
int x   = b[3];                       // O(1) read
b[3]    = 99;                         // O(1) write

// Filling
Arrays.fill(a, -1);                   // [-1,-1,-1,-1,-1]
Arrays.fill(a, 1, 4, 7);              // fill index [1,4) with 7

// Copying
int[] copy  = Arrays.copyOf(b, b.length);       // full copy
int[] slice = Arrays.copyOfRange(b, 2, 5);      // [b[2], b[3], b[4]]
System.arraycopy(b, 0, copy, 0, b.length);      // fastest manual copy

// Sorting
Arrays.sort(b);                                  // O(n log n), in place, dual-pivot quicksort
Arrays.sort(b, 0, 4);                            // sort sub-range [0,4)

// For OBJECT arrays you can pass a Comparator
Integer[] boxed = {3, 1, 4, 1, 5};
Arrays.sort(boxed, (x1, y1) -> y1 - x1);         // descending

// Searching (array MUST be sorted first)
int idx = Arrays.binarySearch(b, 5);             // returns index, or -(insertion_point + 1)

// Comparing & printing
boolean eq = Arrays.equals(a, copy);             // element-wise
String s   = Arrays.toString(b);                 // "[1, 1, 2, 3, ...]"
String s2  = Arrays.deepToString(grid);          // for 2D arrays

// Convert array <-> List
List<Integer> list = Arrays.asList(boxed);       // FIXED-SIZE backed by array — NOT a real ArrayList
List<Integer> mutable = new ArrayList<>(Arrays.asList(boxed));  // safer
Integer[] backToArr = list.toArray(new Integer[0]);

// Convert int[] <-> List<Integer> — primitives need streams (no autoboxing for arrays)
int[] prims = {1, 2, 3};
List<Integer> boxedList = Arrays.stream(prims).boxed().toList();
int[] unboxed = boxedList.stream().mapToInt(Integer::intValue).toArray();

// Iterating
for (int v : b) { /* enhanced for, no index */ }
for (int i = 0; i < b.length; i++) { /* classic, gives index */ }
```

### Under the hood
- A Java array is a contiguous block of memory plus a header storing the length. `arr.length` is a final field.
- Indexing is `address = base + i * sizeof(T)`. That's why it's O(1) — no comparison, just math.
- Bounds checking happens on every access (`ArrayIndexOutOfBoundsException`). The JIT can sometimes eliminate it inside tight loops, but you don't get to opt out manually.

### Gotchas & predict-the-output traps
- **`Arrays.asList(int[])` does NOT do what you think.** Passing a primitive `int[]` gives you `List<int[]>` of size **1** — it treats the array itself as a single element. Use `Arrays.stream(arr).boxed().toList()` for primitives.
- **`Arrays.asList(...)` is fixed-size.** Calling `.add()` on it throws `UnsupportedOperationException`. Wrap with `new ArrayList<>(Arrays.asList(...))` if you need to grow.
- **`int[]` default is 0; `Integer[]` default is `null`.** Iterating an `Integer[]` and unboxing without a null check throws NPE.
- **`==` on arrays compares references**, not contents. Use `Arrays.equals` (1D) or `Arrays.deepEquals` (nested).
- **`array.length` has no parens**; `string.length()` does; `list.size()` does. Easy to mis-type.
- **Sorting a `long[]` and using `(a,b) -> a - b`** can overflow. Use `Long.compare(a, b)` or `Integer.compare(a, b)` for safety.

### Pattern template — Two pointers on a sorted array
```java
public boolean hasPairWithSum(int[] sorted, int target) {
    int l = 0, r = sorted.length - 1;
    while (l < r) {
        int sum = sorted[l] + sorted[r];
        if (sum == target) return true;
        if (sum < target) l++;
        else r--;
    }
    return false;
}
```

### Expected DSA problem types
- **Two-pointer** — *Container With Most Water, Trapping Rain Water, 3Sum*
- **Sliding window** — *Maximum Sum Subarray of Size K, Longest Substring Without Repeating Characters*
- **Prefix sum** — *Subarray Sum Equals K, Range Sum Query*
- **Cyclic sort** — *Find Missing Number, Find All Duplicates in an Array*
- **In-place rearrangement** — *Move Zeroes, Rotate Array*
- **Kadane's algorithm** — *Maximum Subarray*

---

## 2. Dynamic Array (`ArrayList`)

### Etymology
*Dynamic* is Greek *dynamis* — "power, capability." A dynamic array has the *capability* to grow. "ArrayList" is a Java naming choice — under the hood it's still an array.

### Real-world example
A **bookshelf you keep adding to**. When it fills up, you move every book to a bigger shelf. You don't move them on every add — only when you outgrow the current shelf. So most adds are cheap; occasional adds are expensive.

### Problem it solves
Arrays have fixed size. Real problems rarely give you the size upfront.

### Complexity at a glance

| Operation             | Time              | Notes                                      |
|-----------------------|-------------------|--------------------------------------------|
| `get(i)` / `set(i,x)` | O(1)              | Random access preserved                    |
| `add(x)` (at end)     | **Amortized O(1)**| Resize is occasional, copy is O(n)         |
| `add(i, x)`           | O(n)              | Shifts elements right                      |
| `remove(i)`           | O(n)              | Shifts elements left                       |
| `remove(Object)`      | O(n)              | Linear scan + shift                        |
| `contains` / `indexOf`| O(n)              | Linear scan, uses `equals`                 |
| `size()`              | O(1)              | Stored as a field                          |
| **Space**             | O(n)              | Up to ~1.5x slack from growth strategy     |

### Methods cheat sheet
```java
import java.util.*;

List<Integer> list = new ArrayList<>();
List<Integer> preSized = new ArrayList<>(1000);          // reserve capacity, avoids resizes
List<Integer> fromCol  = new ArrayList<>(List.of(1,2,3));// copy from another collection

// Adding
list.add(42);                  // append, amortized O(1)
list.add(0, 99);               // insert at index 0, O(n)
list.addAll(List.of(7, 8, 9)); // append all
list.addAll(1, List.of(5, 6)); // insert all at index 1

// Reading
int x   = list.get(2);
int sz  = list.size();
boolean empty = list.isEmpty();

// Updating
list.set(2, 77);               // replace, returns old value

// Removing
list.remove(0);                // by INDEX, returns removed element
list.remove(Integer.valueOf(42)); // by VALUE — note the boxing trick (see gotchas)
list.removeIf(v -> v % 2 == 0);   // predicate-based, safe during iteration
list.clear();                  // empty the list

// Searching
boolean has = list.contains(7);
int idx     = list.indexOf(7);
int last    = list.lastIndexOf(7);

// Slicing — returns a VIEW, not a copy (mutations propagate)
List<Integer> sub = list.subList(1, 4);

// Conversion
Integer[] arr = list.toArray(new Integer[0]);
int[] prims   = list.stream().mapToInt(Integer::intValue).toArray();

// Sorting (in place)
Collections.sort(list);
list.sort((a, b) -> b - a);     // custom: descending
list.sort(Comparator.reverseOrder());

// Iteration
for (int v : list) { ... }                        // enhanced for
list.forEach(v -> System.out.println(v));         // lambda
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    int v = it.next();
    if (v == 0) it.remove();   // ONLY safe way to remove during iteration
}
```

### Under the hood
- Backed by `Object[]` (`elementData`), with an `int size` field that tracks logical length (≤ array length).
- Initial capacity is 10 on first add (lazy in modern JDKs). When full, new capacity = `oldCapacity + (oldCapacity >> 1)` ≈ 1.5×, then `Arrays.copyOf` copies everything over.
- The 1.5× growth is what makes `add` amortized O(1): the cost of copies across many adds averages to a constant per add.
- `ArrayList` is **not synchronized**. Use `Collections.synchronizedList(...)` or `CopyOnWriteArrayList` for concurrent use.

### Gotchas & predict-the-output traps
- **`list.remove(int)` vs `list.remove(Integer)`** — overload resolution picks `remove(int)` (by index) for primitive `int`, and `remove(Object)` (by value) for boxed `Integer`. To remove the *value* 5, write `list.remove(Integer.valueOf(5))`, not `list.remove(5)`.
- **`Arrays.asList` returns a fixed-size list**, not an `ArrayList`. `.add()` on it throws.
- **`subList` returns a live view.** Mutating either side affects the other. Also, structurally modifying the backing list invalidates the sublist.
- **`ConcurrentModificationException`** — modifying a list with `list.add/remove` while inside a `for-each` loop throws it. Use the iterator's `remove()` or `removeIf`.
- **Diamond inference** — `List<Integer> l = new ArrayList();` (raw on the right) compiles with a warning. Always use `<>`.

### Pattern template — Result accumulation
```java
List<List<Integer>> results = new ArrayList<>();
void backtrack(int[] nums, List<Integer> path, int start) {
    results.add(new ArrayList<>(path));   // SNAPSHOT — never add `path` itself, it'll mutate later
    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        backtrack(nums, path, i + 1);
        path.remove(path.size() - 1);     // undo
    }
}
```
The `new ArrayList<>(path)` line is one of the most missed details in subset/permutation problems.

### Expected DSA problem types
- Anywhere you'd use an array but don't know the output size
- Building results in DFS/BFS where the count varies
- Result accumulation in problems like *Subsets, Permutations, Combinations*

---

## 3. Linked List

### Etymology
*List* is Old English *liste* — originally "a strip, a border, a catalog written on a strip of parchment." *Linked* is straightforward — chained together.

### Real-world example
A **scavenger hunt**. Each clue tells you only where the *next* clue is. You can't skip ahead to clue #5 without reading clues 1 through 4. But adding a new clue is trivial — write a note pointing to the next location, and have the previous clue point to your new note instead.

### Problem it solves
Arrays are slow to insert into the middle (everything shifts). Linked lists make insert/delete O(1) **if you already have a pointer to the spot** — at the cost of losing random access.

### Complexity at a glance

| Operation                  | Time | Notes                                  |
|----------------------------|------|----------------------------------------|
| `addFirst` / `addLast`     | O(1) |                                        |
| `removeFirst` / `removeLast` | O(1) |                                      |
| `get(i)` / `set(i, x)`     | O(n) | Walks from nearest end                 |
| `add(i, x)` / `remove(i)`  | O(n) | Walk + relink                          |
| `contains(x)`              | O(n) |                                        |
| `size()`                   | O(1) | Stored as a field                      |
| **Space**                  | O(n) | Plus 2 pointer overhead per node       |

### Methods cheat sheet
```java
import java.util.*;

// Hand-rolled singly linked node — what you'll actually write in interviews
class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
    ListNode(int val, ListNode next) { this.val = val; this.next = next; }
}

// Java's built-in is a doubly-linked list, also implements Deque
LinkedList<Integer> ll = new LinkedList<>();

// Adding
ll.addFirst(1);          // O(1) — head
ll.addLast(2);           // O(1) — tail
ll.add(3);               // alias for addLast
ll.add(1, 99);           // insert at index 1, O(n)
ll.offer(4);             // Queue interface — add at tail
ll.push(5);              // Deque interface — add at HEAD (yes, opposite of intuition for Stack)

// Reading
int head = ll.getFirst();    // throws NoSuchElementException if empty
int tail = ll.getLast();
int peek = ll.peek();        // returns null if empty (Queue interface)
int peekF= ll.peekFirst();
int peekL= ll.peekLast();
int at   = ll.get(2);        // O(n)

// Removing
ll.removeFirst();        // O(1), throws if empty
ll.removeLast();         // O(1)
ll.poll();               // returns null if empty (Queue interface)
ll.pollFirst(); ll.pollLast();
ll.pop();                // Deque interface — removes HEAD
ll.remove(Integer.valueOf(99));   // first occurrence by VALUE

// Iteration (forward & backward)
Iterator<Integer> fwd  = ll.iterator();
Iterator<Integer> back = ll.descendingIterator();
```

### Under the hood
- Java's `LinkedList` is a **doubly linked list** with `first` and `last` pointers and a `size` field.
- Each node holds `prev`, `next`, and `item` — three references plus the value, so memory overhead per element is significant (~40+ bytes per Integer node on a 64-bit JVM).
- `get(i)` is "smart": it walks from `first` if `i < size/2`, else from `last`. Still O(n).
- Pointer chasing destroys CPU cache locality — this is the real reason `ArrayList` usually wins benchmarks.

### Gotchas & predict-the-output traps
- **`push`/`pop` go to the HEAD**, not the tail (because `Deque` defines stack semantics on the front). `offer`/`poll` use Queue semantics — also at head/tail respectively but for FIFO.
- **`LinkedList` implements both `Queue` and `Deque` AND `List`.** You'll see all three method styles mixed in code, and they often do the same thing through different names.
- **`getFirst` throws on empty; `peek` returns null.** Mixing them is a common bug.
- **NullPointerException when traversing** — always check `node != null` before `node.next`.
- **Losing the head reference** — if you reassign `head = head.next` and never saved the original, you can't return it.
- **In Java's built-in `LinkedList`, indexed access is O(n)** even though it has a `get(i)` — never use it in a tight loop.

### Pattern template — Reverse a singly linked list (iterative)
```java
public ListNode reverse(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next;   // save before overwriting
        curr.next = prev;            // flip pointer
        prev = curr;                 // advance prev
        curr = next;                 // advance curr
    }
    return prev;   // new head
}
```

### Pattern template — Floyd's cycle detection (fast & slow)
```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### Expected DSA problem types
This is its own LeetCode category. Patterns:
- **Reverse a linked list** — *Reverse Linked List, Reverse Nodes in K-Group*
- **Fast & slow pointers (Floyd's)** — *Linked List Cycle, Find Middle of Linked List*
- **Merge two lists** — *Merge Two Sorted Lists, Merge K Sorted Lists*
- **Remove Nth from end** (two pointers, gap of N)
- **Add two numbers** (digits stored in reverse)
- **LRU Cache** — combines doubly linked list + HashMap

---

## 4. Stack

### Etymology
From Old Norse *stakkr* — "a haystack, a pile." The English meaning of "a vertical pile of things" came from the shape of stacked hay.

### Real-world example
**Stack of plates in a cafeteria**. You put a clean plate on top, you take a plate from the top. The bottom plate has been there longest. **LIFO** — Last In, First Out. You physically *can't* take the bottom plate without removing everything above it first.

### Problem it solves
Anywhere you need to **remember things in reverse order** of when you saw them. Function call frames, undo histories, matching parentheses, backtracking — they all need "give me the most recent thing I haven't dealt with yet."

### Complexity at a glance

| Operation         | Time | Notes                          |
|-------------------|------|--------------------------------|
| `push(x)`         | O(1) | Add to top                     |
| `pop()`           | O(1) | Remove and return top          |
| `peek()`          | O(1) | Look at top, no removal        |
| `isEmpty()`       | O(1) |                                |
| `contains(x)`     | O(n) | Linear scan, rarely used       |
| **Space**         | O(n) |                                |

### Methods cheat sheet
```java
import java.util.*;

// MODERN — use ArrayDeque for stack semantics
Deque<Integer> stack = new ArrayDeque<>();

stack.push(1);          // O(1) — adds to head
stack.push(2);
stack.push(3);

int top  = stack.peek();  // 3, no removal
int gone = stack.pop();   // 3, removed
int sz   = stack.size();
boolean empty = stack.isEmpty();

// Iterating a Deque-as-stack goes TOP TO BOTTOM (head-first)
for (int v : stack) System.out.println(v);   // prints 2, 1

// LEGACY — avoid in new code
Stack<Integer> legacy = new Stack<>();   // extends Vector, fully synchronized → slow
legacy.push(1); legacy.pop(); legacy.peek();
// Vector iteration order is BOTTOM to TOP — opposite of ArrayDeque. This trips people up.
```

### Under the hood
- `ArrayDeque` is a circular array with two pointers (head, tail). When full, it doubles. No node allocation per push, so it's much faster than `LinkedList` or legacy `Stack`.
- The legacy `java.util.Stack` extends `Vector`, which extends `AbstractList`. Every method is `synchronized`, even when you're single-threaded. It also exposes `List` methods like `get(i)` — semantically broken for a stack.

### Gotchas & predict-the-output traps
- **Iteration order differs:** `ArrayDeque` iterates head-first (top of stack first). `java.util.Stack` (which extends `Vector`) iterates bottom-up. Mixing them in test code is a classic trap.
- **`ArrayDeque.push`/`pop`/`peek` all act on the HEAD**, even though intuitively a "stack top" feels like the tail. This is consistent with `Deque` contract.
- **Don't put `null` into `ArrayDeque`.** It throws `NullPointerException`. Other deques (like `LinkedList`) allow nulls — but then `peek()` returning `null` becomes ambiguous (empty? or null element?).
- **Recursion uses the call stack.** Deep recursion → `StackOverflowError`. Sometimes the fix is converting recursion into an explicit `Deque<Frame>`.

### Pattern template — Monotonic stack (Next Greater Element)
```java
public int[] nextGreater(int[] nums) {
    int n = nums.length;
    int[] res = new int[n];
    Arrays.fill(res, -1);
    Deque<Integer> stack = new ArrayDeque<>();   // stores INDICES
    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
            res[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return res;
}
```

### Pattern template — Balanced parentheses
```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pair = Map.of(')','(', ']','[', '}','{');
    for (char c : s.toCharArray()) {
        if (pair.containsValue(c)) stack.push(c);          // opener
        else if (stack.isEmpty() || stack.pop() != pair.get(c)) return false;
    }
    return stack.isEmpty();
}
```

### Expected DSA problem types
- **Balanced parentheses** — *Valid Parentheses, Minimum Remove to Make Valid*
- **Monotonic stack** — *Next Greater Element, Daily Temperatures, Largest Rectangle in Histogram*
- **Expression evaluation** — *Evaluate Reverse Polish Notation, Basic Calculator*
- **Backtracking simulation** — *Decode String*
- **DFS iterative** — any tree/graph DFS without recursion
- **Min Stack** (classic system design-y question)

---

## 5. Queue and Deque

### Etymology
*Queue* is from Latin *cauda* — "tail." Came through French where *queue* still means "tail" or "ponytail." A British queue at a bus stop is literally a "tail" of people.

*Deque* is short for **D**ouble-**E**nded **Que**ue, pronounced "deck."

### Real-world example
A **line at a coffee shop**. First person in line gets served first. **FIFO** — First In, First Out. You can't cut to the front, and the barista can't randomly serve someone in the middle.

A **deque** is a line where people can be added or removed from *either* end — like a passenger train where people board and exit from both the front and back doors.

### Problem it solves
- **Queue**: process things in arrival order. Critical for BFS, scheduling, buffering.
- **Deque**: when you need fast access to *both* ends — sliding window problems where you add to the back and remove from either side.

### Complexity at a glance

| Operation                          | Time | Notes                            |
|------------------------------------|------|----------------------------------|
| `offer` / `poll` / `peek`          | O(1) | Queue: tail/head/head            |
| `offerFirst` / `offerLast`         | O(1) | Deque                            |
| `pollFirst` / `pollLast`           | O(1) | Deque                            |
| `peekFirst` / `peekLast`           | O(1) | Deque                            |
| `contains(x)`                      | O(n) |                                  |
| **Space**                          | O(n) |                                  |

### Methods cheat sheet
```java
import java.util.*;

// QUEUE — process in arrival order
Queue<Integer> q = new ArrayDeque<>();

q.offer(1);              // add at tail; returns true (vs add() which throws on capacity-bounded)
q.offer(2);
int head = q.peek();     // 1, returns null if empty
int gone = q.poll();     // 1, returns null if empty
int sz   = q.size();
boolean empty = q.isEmpty();

// add/remove/element are the throwing variants — avoid in DSA, prefer offer/poll/peek
// q.add(x);  q.remove();  q.element();   // throw on failure

// DEQUE — both ends
Deque<Integer> dq = new ArrayDeque<>();
dq.offerFirst(1); dq.offerLast(2); dq.offerLast(3);   // [1,2,3]
dq.peekFirst();  // 1
dq.peekLast();   // 3
dq.pollFirst();  // 1
dq.pollLast();   // 3

// Stack-style methods on a Deque (act on HEAD)
dq.push(99);     // = offerFirst(99)
dq.pop();        // = pollFirst()
dq.peek();       // = peekFirst()
```

### The throw-vs-return-null cheat sheet
| Family   | On success | On failure (empty/full) |
|----------|------------|-------------------------|
| Throwing | `add`, `remove`, `element`, `getFirst`, `getLast` | throws exception |
| Null-returning | `offer`, `poll`, `peek`, `peekFirst`, `pollFirst` | returns `null` / `false` |

In DSA code, **always use the null-returning variants** so empty checks read cleanly.

### Under the hood
- `ArrayDeque` is a circular buffer (resizable array with wrap-around indices). No per-element node allocation. Doubles on full.
- `LinkedList` also implements `Deque` but allocates a node per element — slower in practice.
- `PriorityQueue` implements `Queue` but is NOT FIFO — it orders by priority. Don't confuse the interface name with FIFO behavior.

### Gotchas & predict-the-output traps
- **`ArrayDeque` does not allow null elements.** Inserting null throws NPE.
- **`Queue` does NOT mean FIFO.** `PriorityQueue` implements `Queue` but pops in priority order — interview trap.
- **`stack.push` and `deque.push` both push to HEAD** — looks counterintuitive when you visualize a deque as left-to-right.
- **`LinkedList` is also a `Queue` and a `Deque`**, but slower than `ArrayDeque` — favor `ArrayDeque` unless you need null support or list semantics.

### Pattern template — BFS on a graph
```java
public int bfsShortestPath(Map<Integer,List<Integer>> graph, int src, int dst) {
    Queue<int[]> q = new ArrayDeque<>();   // [node, distance]
    Set<Integer> visited = new HashSet<>();
    q.offer(new int[]{src, 0});
    visited.add(src);
    while (!q.isEmpty()) {
        int[] cur = q.poll();
        if (cur[0] == dst) return cur[1];
        for (int nei : graph.getOrDefault(cur[0], List.of())) {
            if (visited.add(nei)) q.offer(new int[]{nei, cur[1] + 1});
        }
    }
    return -1;
}
```

### Pattern template — Monotonic deque (Sliding Window Maximum)
```java
public int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> dq = new ArrayDeque<>();   // stores INDICES, values at indices are decreasing
    int[] res = new int[nums.length - k + 1];
    for (int i = 0; i < nums.length; i++) {
        // shrink from front when out of window
        while (!dq.isEmpty() && dq.peekFirst() <= i - k) dq.pollFirst();
        // shrink from back to keep monotonic decreasing
        while (!dq.isEmpty() && nums[dq.peekLast()] < nums[i]) dq.pollLast();
        dq.offerLast(i);
        if (i >= k - 1) res[i - k + 1] = nums[dq.peekFirst()];
    }
    return res;
}
```

### Expected DSA problem types
- **BFS** — *Number of Islands, Word Ladder, Rotting Oranges, Binary Tree Level Order Traversal*
- **Topological sort** (Kahn's algorithm) — *Course Schedule*
- **Multi-source BFS** — *Walls and Gates, 01 Matrix*
- **Sliding window maximum** (uses deque) — *Sliding Window Maximum*
- **Monotonic deque** — *Shortest Subarray with Sum at Least K*

---

## 6. Priority Queue (Heap)

### Etymology
*Heap* is Old English *heap* — "a pile, a mass." The data structure is named after the *visual* of a binary heap — a pile-like tree where the heaviest (or lightest) thing sits on top. *Priority queue* is a modern compound — a queue ordered by priority, not arrival.

### Real-world example
A **hospital ER triage**. People don't get seen in order of arrival — they get seen by severity. The patient with the gunshot wound jumps the line. A new arrival gets inserted at the right priority spot, and the next-most-critical patient is always next.

### Problem it solves
"Give me the smallest (or largest) element right now, and let me keep adding new elements." A sorted list would solve it, but inserts would be O(n). A heap gives O(log n) insert and O(1) peek-min.

### Complexity at a glance

| Operation             | Time      | Notes                                  |
|-----------------------|-----------|----------------------------------------|
| `offer(x)` / `add`    | O(log n)  | Sift up                                |
| `poll()` / `remove()` | O(log n)  | Sift down                              |
| `peek()`              | O(1)      | Top of heap                            |
| `contains(x)`         | O(n)      | Linear scan — common surprise          |
| `remove(Object)`      | O(n)      | Linear scan + sift                     |
| Heapify from array    | O(n)      | Bottom-up build via constructor        |
| **Space**             | O(n)      |                                        |

### Methods cheat sheet
```java
import java.util.*;

// MIN-HEAP (default) — smallest element on top
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(2);
minHeap.offer(8);
minHeap.peek();      // 2, O(1)
minHeap.poll();      // 2, O(log n)

// MAX-HEAP — three idiomatic ways
PriorityQueue<Integer> maxA = new PriorityQueue<>(Collections.reverseOrder());
PriorityQueue<Integer> maxB = new PriorityQueue<>((a, b) -> b - a);          // OK for ints, NOT for longs
PriorityQueue<Integer> maxC = new PriorityQueue<>((a, b) -> Integer.compare(b, a));  // safest

// CUSTOM COMPARATOR — heap of int[] keyed on the second element
PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));

// Multi-key sort: by frequency asc, then by value desc on ties
PriorityQueue<int[]> tiebreak = new PriorityQueue<>(
    Comparator.<int[]>comparingInt(a -> a[0]).thenComparing((a, b) -> b[1] - a[1])
);

// HEAPIFY — build from existing collection in O(n)
PriorityQueue<Integer> built = new PriorityQueue<>(List.of(5, 2, 8, 1, 9));

// Common operations
int sz = pq.size();
boolean empty = pq.isEmpty();
pq.clear();

// Iterating — NOT in sorted order. Iteration order is INTERNAL HEAP ORDER (basically arbitrary).
// To get sorted output, repeatedly poll().
```

### Under the hood
- A **binary heap** stored as an array. For index `i`:
  - parent: `(i - 1) / 2`
  - left child: `2i + 1`
  - right child: `2i + 2`
- **Heap property** (min-heap): every parent ≤ both children. The root is always the minimum.
- `offer` appends at the end and "sifts up" until the heap property is restored — O(log n).
- `poll` swaps root with last, removes last, "sifts down" — O(log n).
- Building from an unordered array is O(n) using bottom-up sift-down (Floyd's algorithm), not O(n log n) as you'd expect from inserting one-by-one.

### Gotchas & predict-the-output traps
- **Iteration is NOT sorted.** `for (int x : pq)` walks the underlying array in heap layout. To traverse in order, repeatedly poll into a list (destructively) or copy first.
- **`(a, b) -> b - a` overflows** when `a` and `b` are far apart and one is `Integer.MIN_VALUE`. Use `Integer.compare(b, a)`. Same for longs — *especially* longs, since `b - a` silently truncates to int when both are int and overflows when long.
- **`contains` and `remove(Object)` are O(n)**, not O(log n). If you need fast key updates, use `TreeMap` or maintain a HashMap on the side ("indexed heap").
- **No max-heap by default** — pass `Collections.reverseOrder()` or a comparator.
- **Comparator must be consistent.** A non-transitive comparator silently corrupts the heap; failures look like "the wrong element was returned."
- **For complex types, you must define `Comparator` or implement `Comparable`.** Forgetting throws `ClassCastException` at first `offer`.

### Pattern template — Top K largest elements
```java
public List<Integer> topK(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();   // min-heap of size k
    for (int n : nums) {
        minHeap.offer(n);
        if (minHeap.size() > k) minHeap.poll();    // evict smallest
    }
    return new ArrayList<>(minHeap);   // unordered, but contains the top k
}
```

### Pattern template — Two-heap median tracker
```java
class MedianFinder {
    // lo holds smaller half (max-heap), hi holds larger half (min-heap)
    PriorityQueue<Integer> lo = new PriorityQueue<>(Collections.reverseOrder());
    PriorityQueue<Integer> hi = new PriorityQueue<>();

    public void addNum(int num) {
        lo.offer(num);
        hi.offer(lo.poll());           // funnel max of lo into hi
        if (hi.size() > lo.size()) lo.offer(hi.poll());   // rebalance
    }
    public double findMedian() {
        return lo.size() > hi.size() ? lo.peek() : (lo.peek() + hi.peek()) / 2.0;
    }
}
```

### Expected DSA problem types
This is one of the **highest-frequency** structures in interviews — Citi/Karat-style especially loves it.
- **Top-K problems** — *Top K Frequent Elements, Kth Largest Element in an Array*
- **K-way merge** — *Merge K Sorted Lists, Smallest Range Covering Elements from K Lists*
- **Two heaps** — *Find Median from Data Stream, Sliding Window Median*
- **Scheduling** — *Task Scheduler, Meeting Rooms II*
- **Dijkstra's shortest path**
- **Prim's MST**

---

## 7. Hash Table (`HashMap`, `HashSet`)

### Etymology
*Hash* is from French *hacher* — "to chop, to mince." Same root as "hatchet" and the breakfast dish "hash browns" — chopped potatoes. The hash *function* "chops up" your key into a number.

*Map* is from Latin *mappa* — originally "a cloth or napkin," then "a chart drawn on cloth." The data structure maps keys to values the way a paper map maps places to coordinates.

### Real-world example
A **library card catalog** (or its modern descendant, a contact list on your phone). You don't search every shelf for a book — you look up the title, get a code, and walk straight to that shelf. The catalog *maps* a title to a location.

A **hash function** is like a function that takes "John Smith" and tells you "drawer #47, slot #3" — directly, without any searching.

### Problem it solves
"I want to look something up by name (or any key) in O(1) time, not O(n)." This is arguably the most important data structure ever invented. It transforms search from a linear walk into a direct jump.

### Complexity at a glance

| Operation                | Average | Worst (Java 8+) | Notes                          |
|--------------------------|---------|------------------|--------------------------------|
| `put(k, v)`              | O(1)    | O(log n)         | Tree-bin fallback ≥8 collisions|
| `get(k)`                 | O(1)    | O(log n)         |                                |
| `containsKey` / `containsValue` | O(1) / O(n) | O(log n) / O(n) | Value scan is linear        |
| `remove(k)`              | O(1)    | O(log n)         |                                |
| Iteration                | O(n + capacity) |          | Includes empty buckets         |
| **Space**                | O(n)    |                  | Plus load-factor slack (~25%)  |

### Methods cheat sheet
```java
import java.util.*;

// HASHMAP — key/value
Map<String, Integer> map = new HashMap<>();
Map<String, Integer> sized = new HashMap<>(64);             // initial capacity
Map<String, Integer> tuned = new HashMap<>(64, 0.75f);      // capacity + load factor

// Putting
map.put("alice", 30);
map.putIfAbsent("alice", 99);   // does NOT overwrite, returns existing (30)
Integer prev = map.put("alice", 31);  // returns previous value (30) or null

// Getting
int  a   = map.get("alice");                  // returns null if absent (autoboxing → NPE risk on int)
int  d   = map.getOrDefault("bob", 0);        // your most-used DSA method
boolean has = map.containsKey("alice");
boolean hasV= map.containsValue(31);          // O(n)

// Deleting
map.remove("alice");                          // returns the removed value or null
map.remove("alice", 99);                      // only removes if value matches (atomic)

// Compute / merge — the power tools
map.computeIfAbsent("c", k -> 0);             // insert default if missing
map.computeIfPresent("a", (k, v) -> v + 1);   // update only if present
map.compute("a", (k, v) -> v == null ? 1 : v + 1);  // upsert
map.merge("a", 1, Integer::sum);              // upsert with combiner — count++ in one line

// Pattern: graph adjacency list
Map<Integer, List<Integer>> graph = new HashMap<>();
graph.computeIfAbsent(u, k -> new ArrayList<>()).add(v);

// Pattern: frequency map
Map<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);

// Iteration
for (var e : map.entrySet()) { String k = e.getKey(); int v = e.getValue(); }
for (String k : map.keySet()) { ... }
for (int v : map.values()) { ... }
map.forEach((k, v) -> System.out.println(k + "=" + v));

// HASHSET — keys only
Set<Integer> seen = new HashSet<>();
seen.add(42);                  // returns true if new
seen.contains(42);
seen.remove(42);

// Set operations
Set<Integer> a = new HashSet<>(List.of(1,2,3));
Set<Integer> b = new HashSet<>(List.of(2,3,4));
Set<Integer> union = new HashSet<>(a); union.addAll(b);          // {1,2,3,4}
Set<Integer> inter = new HashSet<>(a); inter.retainAll(b);       // {2,3}
Set<Integer> diff  = new HashSet<>(a); diff.removeAll(b);        // {1}
```

### Under the hood
- An array of "buckets" (`Node<K,V>[] table`). A key's hash determines which bucket: `index = (n - 1) & hash`.
- Java 8+ uses an enhanced hash mixing: `hash = (h = key.hashCode()) ^ (h >>> 16)` — XORs the upper 16 bits down to spread entropy.
- Collisions go into a **linked list** in the bucket. When a bucket reaches **TREEIFY_THRESHOLD = 8** entries AND table size ≥ 64, it converts to a **red-black tree**, giving O(log n) worst case instead of O(n).
- **Load factor** (default 0.75): when `size > capacity * 0.75`, the table doubles and all entries are rehashed.
- **Initial capacity** is rounded up to the next power of 2 (so 17 → 32). Defaults to 16 in modern JDKs.

### Gotchas & predict-the-output traps
- **Mutating a key after insertion breaks the map.** If a key's `hashCode` changes, the entry is "lost" — `get` looks in a different bucket. Use immutable keys (`String`, `Integer`, records, or DTOs without setters).
- **`equals` and `hashCode` must be consistent.** If `a.equals(b)` is true, `a.hashCode() == b.hashCode()` MUST be true. Violating this corrupts every Java collection that uses hashing. Common bug: overriding `equals` and forgetting `hashCode`.
- **Auto-unboxing NPE.** `int x = map.get("missing")` throws NPE because `get` returns `null` (boxed `Integer`), and unboxing null throws. Use `getOrDefault` or check first.
- **Iteration order is not insertion order**, not sorted, **and not stable across JVM versions**. Don't assume it.
- **`containsKey(null)` and `null` values are allowed in `HashMap`.** `Hashtable` and `ConcurrentHashMap` reject nulls. Common surprise.
- **`HashSet` is internally a `HashMap` with a dummy value.** Same gotchas apply.
- **`putAll` does NOT clear first** — it merges, overwriting on conflict.

### `equals`/`hashCode` contract — the rule
```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;   // pattern variable, Java 16+
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() {
        return Objects.hash(x, y);   // never just `return x ^ y` for general use
    }
}
```
**Rule**: equal objects must produce equal hash codes. Unequal objects *should* (but aren't required to) produce different hash codes — collisions hurt performance, not correctness.

### Pattern template — Two Sum
```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> idx = new HashMap<>();   // value -> index
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (idx.containsKey(complement)) return new int[]{idx.get(complement), i};
        idx.put(nums[i], i);
    }
    return new int[0];
}
```

### Pattern template — Group Anagrams
```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(groups.values());
}
```

### Expected DSA problem types
The single most-used structure in LeetCode Easy/Medium.
- **Two Sum** and every variant
- **Frequency counting** — *Group Anagrams, Top K Frequent Elements*
- **Seen/visited tracking** — *Contains Duplicate, Longest Consecutive Sequence*
- **Caching/memoization** in DP
- **Sliding window with character counts** — *Longest Substring Without Repeating Characters, Minimum Window Substring*
- **Adjacency lists** for graphs
- **Two-sum-style "complement"** — *Two Sum, 3Sum, 4Sum*

---

## 8. LinkedHashMap / LinkedHashSet

### Etymology
The "linked" prefix tells you how it's implemented — a hash table *plus* a doubly linked list weaving through the entries to remember insertion order.

### Real-world example
A **chronological diary with an index**. You can flip to any date instantly (the index = hash table), but if you read the entries in physical order, they're in the order you wrote them (the linked list).

### Problem it solves
"Hash map speed, but I want to iterate in insertion order." Or: "I want LRU cache behavior — quickly evict the oldest entry."

### Complexity at a glance

| Operation       | Time | Notes                                |
|-----------------|------|--------------------------------------|
| `put` / `get`   | O(1) | Same as HashMap, slight constant overhead from list maintenance |
| `remove`        | O(1) |                                      |
| Iteration       | O(n) | Walks the linked list — no empty-bucket overhead, unlike HashMap |
| **Space**       | O(n) | Plus 2 extra pointers per entry      |

### Methods cheat sheet
```java
import java.util.*;

// Default — INSERTION-ORDER iteration
Map<String, Integer> ordered = new LinkedHashMap<>();
ordered.put("c", 3); ordered.put("a", 1); ordered.put("b", 2);
ordered.keySet();   // [c, a, b] — insertion order, NOT sorted

// ACCESS-ORDER mode — moves accessed entry to the tail
// Constructor: (initialCapacity, loadFactor, accessOrder=true)
Map<Integer, String> accessOrdered = new LinkedHashMap<>(16, 0.75f, true);
accessOrdered.put(1, "a"); accessOrdered.put(2, "b"); accessOrdered.put(3, "c");
accessOrdered.get(1);                  // touching key 1 moves it to the END
// Iteration order is now: 2, 3, 1

// LRU CACHE — override removeEldestEntry to auto-evict
class LRU<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LRU(int cap) {
        super(cap, 0.75f, true);   // access-order mode
        this.capacity = cap;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;   // returning true tells the map to evict 'eldest'
    }
}
LRU<Integer, String> cache = new LRU<>(3);
cache.put(1,"a"); cache.put(2,"b"); cache.put(3,"c");
cache.get(1);            // 1 becomes most recently used
cache.put(4,"d");        // capacity exceeded → evicts 2 (least recently used)
```

### Under the hood
- `LinkedHashMap` extends `HashMap`, adding `before` and `after` pointers in each entry.
- Inserts always append to the doubly linked list. With `accessOrder=true`, every `get`/`put` also re-links the entry to the tail.
- `removeEldestEntry` is called on every `put` after insertion; returning `true` evicts the head of the linked list.

### Gotchas & predict-the-output traps
- **Access-order is OFF by default.** Without the 3-arg constructor, you only get insertion-order — no LRU behavior.
- **`get` mutates structure when `accessOrder=true`.** That means `get` can throw `ConcurrentModificationException` mid-iteration. It also means *reading* a shared `LinkedHashMap` is unsafe across threads — the access-order link rewrite is not atomic.
- **Eviction happens AFTER insert**, so during the very call to `put`, the map briefly holds `capacity + 1` entries.
- **`putIfAbsent` does NOT count as access** in some JDK versions. Verify before relying on it for LRU.

### Pattern template — Hand-rolled LRU (HashMap + Doubly Linked List)
This is the version interviewers usually want, even though `LinkedHashMap` would solve it in 5 lines.
```java
class LRUCache {
    private static class Node {
        int key, val;
        Node prev, next;
        Node(int k, int v) { key = k; val = v; }
    }
    private final int cap;
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0, 0);   // dummy head
    private final Node tail = new Node(0, 0);   // dummy tail

    public LRUCache(int capacity) {
        this.cap = capacity;
        head.next = tail;
        tail.prev = head;
    }
    public int get(int key) {
        Node n = map.get(key);
        if (n == null) return -1;
        moveToFront(n);
        return n.val;
    }
    public void put(int key, int value) {
        Node n = map.get(key);
        if (n != null) {
            n.val = value;
            moveToFront(n);
            return;
        }
        n = new Node(key, value);
        map.put(key, n);
        addFront(n);
        if (map.size() > cap) {
            Node lru = tail.prev;
            remove(lru);
            map.remove(lru.key);
        }
    }
    private void addFront(Node n) {
        n.next = head.next; n.prev = head;
        head.next.prev = n; head.next = n;
    }
    private void remove(Node n) {
        n.prev.next = n.next;
        n.next.prev = n.prev;
    }
    private void moveToFront(Node n) { remove(n); addFront(n); }
}
```

### Expected DSA problem types
- **LRU Cache** — classic interview question; can be solved with `LinkedHashMap` directly or implemented from scratch (HashMap + DoublyLinkedList).
- **First non-repeating character** in a stream

---

## 9. TreeMap / TreeSet (Red-Black Tree)

### Etymology
*Tree* is Old English *treow* — the actual plant. Computer scientists borrowed the metaphor for any branching hierarchy. *Red-Black* is just the coloring rule used to keep the tree balanced.

### Real-world example
A **dictionary**. Words are sorted alphabetically. You can find a word by binary searching, you can find all words starting with "pre-", and you can ask "what's the word that comes right after 'apple' alphabetically?" — none of which a hash map can do.

### Problem it solves
You need hash-map-like fast lookup *plus* one of: sorted iteration, range queries, "next key greater than X," or "key just below X."

### Complexity at a glance

| Operation                                    | Time     | Notes                          |
|----------------------------------------------|----------|--------------------------------|
| `put` / `get` / `remove` / `containsKey`     | O(log n) |                                |
| `firstKey` / `lastKey`                       | O(log n) | Walk to leftmost / rightmost   |
| `floorKey` / `ceilingKey` / `higherKey` / `lowerKey` | O(log n) |                       |
| `subMap` / `headMap` / `tailMap`             | O(log n) | Returns a VIEW, not a copy     |
| Sorted iteration                             | O(n)     | In-order traversal             |
| **Space**                                    | O(n)     |                                |

### Methods cheat sheet
```java
import java.util.*;

TreeMap<Integer, String> tm = new TreeMap<>();
tm.put(5, "five"); tm.put(2, "two"); tm.put(8, "eight"); tm.put(3, "three");

// Endpoints
tm.firstKey();                // 2 — smallest
tm.lastKey();                 // 8 — largest
tm.firstEntry();              // Map.Entry(2 -> "two")
tm.lastEntry();               // Map.Entry(8 -> "eight")
tm.pollFirstEntry();          // remove and return smallest
tm.pollLastEntry();           // remove and return largest

// Neighbor lookups — the killer feature
tm.ceilingKey(6);             // 8  — smallest key >= 6
tm.floorKey(6);               // 5  — largest  key <= 6
tm.higherKey(5);              // 8  — smallest key STRICTLY > 5
tm.lowerKey(5);               // 3  — largest  key STRICTLY < 5

// Range views (live views, not copies)
SortedMap<Integer, String> headView = tm.headMap(5);          // keys < 5
SortedMap<Integer, String> tailView = tm.tailMap(5);          // keys >= 5
SortedMap<Integer, String> subView  = tm.subMap(3, 7);        // [3, 7)
NavigableMap<Integer, String> closed = tm.subMap(3, true, 7, true); // [3, 7] inclusive

// Reverse view
NavigableMap<Integer, String> desc = tm.descendingMap();
NavigableSet<Integer> descKeys = tm.descendingKeySet();

// Custom ordering
TreeMap<String, Integer> caseInsensitive = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);

// TREESET — same operations, no values
TreeSet<Integer> ts = new TreeSet<>();
ts.add(5); ts.add(2); ts.add(8);
ts.first();  ts.last();
ts.ceiling(6); ts.floor(6);  ts.higher(5); ts.lower(5);
ts.subSet(3, 7);
```

### Under the hood
- A **Red-Black Tree** — a self-balancing binary search tree where each node is red or black, and balance invariants guarantee tree height stays ≤ 2 log n.
- Because it's BST-based, *in-order traversal* yields sorted iteration for free.
- Comparator (or natural `Comparable` ordering) defines equality — `tm.containsKey(k)` returns true when `compare(k, existingKey) == 0`, NOT when `equals` returns true. Subtle gotcha when ordering doesn't match `equals`.

### Gotchas & predict-the-output traps
- **TreeMap uses `compare`, not `equals`, for key identity.** With `String.CASE_INSENSITIVE_ORDER`, putting `"hello"` then `"HELLO"` overwrites — only one entry survives.
- **Comparator must be total and consistent.** Returning 0 for distinct keys collapses them into one entry.
- **`subMap(from, to)` is `[from, to)` — inclusive-exclusive.** Easy to off-by-one.
- **Views (headMap/tailMap/subMap) are LIVE.** Modifying through the view modifies the original. Inserting outside the view's range throws `IllegalArgumentException`.
- **No `null` keys allowed** (in natural ordering — null can't be compared). Values can be null.
- **TreeMap is slower than HashMap by a 3–10x constant factor** for plain put/get. Only use it when you need ordering features.

### Pattern template — Calendar / interval booking
```java
class MyCalendar {
    private final TreeMap<Integer, Integer> bookings = new TreeMap<>();   // start -> end

    public boolean book(int start, int end) {
        Integer prevStart = bookings.floorKey(start);
        Integer nextStart = bookings.ceilingKey(start);
        // overlap with previous booking?
        if (prevStart != null && bookings.get(prevStart) > start) return false;
        // overlap with next booking?
        if (nextStart != null && nextStart < end) return false;
        bookings.put(start, end);
        return true;
    }
}
```

### Expected DSA problem types
- **Range queries** — *My Calendar I, II, III*
- **Floor / ceiling lookups** — *Find Right Interval*
- **Sliding window with sorted state** — *Sliding Window Median (alternative to two heaps)*
- **Sweep line algorithms**
- **Stock market / event scheduling problems**

---

## 10. Tree (General) and Binary Search Tree

### Etymology
Same *treow* as above. The CS metaphor — root, branches, leaves — maps so cleanly that the term stuck immediately.

### Real-world example
A **family tree** (general tree, any number of children). A **decision flowchart** where every question has exactly two answers — yes/no — is a **binary tree**. A **company org chart** where each manager has multiple reports is a general tree.

### Problem it solves
Hierarchies. Anything that branches. Plus, with the *binary search* property (left subtree < root < right subtree), you get O(log n) search/insert/delete on a balanced tree.

### Complexity at a glance

| Operation                  | Balanced BST | Unbalanced (worst) |
|----------------------------|--------------|--------------------|
| Search / Insert / Delete   | O(log n)     | O(n) — degenerates to a list |
| Min / Max                  | O(log n)     | O(n)               |
| Traversal (any order)      | O(n)         | O(n)               |
| **Space**                  | O(n)         | O(n)               |

### Methods cheat sheet — write the node yourself
```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
    TreeNode(int val, TreeNode l, TreeNode r) { this.val = val; left = l; right = r; }
}

// BST insert (recursive)
TreeNode insert(TreeNode root, int val) {
    if (root == null) return new TreeNode(val);
    if (val < root.val)      root.left  = insert(root.left, val);
    else if (val > root.val) root.right = insert(root.right, val);
    return root;   // duplicates ignored
}

// BST search
TreeNode search(TreeNode root, int target) {
    if (root == null || root.val == target) return root;
    return target < root.val ? search(root.left, target) : search(root.right, target);
}
```

### The four canonical traversals
```java
// PREORDER — root, left, right (good for serialization, copying)
void preorder(TreeNode n, List<Integer> out) {
    if (n == null) return;
    out.add(n.val);
    preorder(n.left, out);
    preorder(n.right, out);
}

// INORDER — left, root, right (yields sorted order on a BST)
void inorder(TreeNode n, List<Integer> out) {
    if (n == null) return;
    inorder(n.left, out);
    out.add(n.val);
    inorder(n.right, out);
}

// POSTORDER — left, right, root (good for deletion, expression trees)
void postorder(TreeNode n, List<Integer> out) {
    if (n == null) return;
    postorder(n.left, out);
    postorder(n.right, out);
    out.add(n.val);
}

// LEVEL-ORDER (BFS) — top-down, level by level
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Queue<TreeNode> q = new ArrayDeque<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int sz = q.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < sz; i++) {
            TreeNode n = q.poll();
            level.add(n.val);
            if (n.left  != null) q.offer(n.left);
            if (n.right != null) q.offer(n.right);
        }
        res.add(level);
    }
    return res;
}

// ITERATIVE INORDER — using an explicit stack (interview favorite)
List<Integer> iterativeInorder(TreeNode root) {
    List<Integer> res = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;
    while (curr != null || !stack.isEmpty()) {
        while (curr != null) { stack.push(curr); curr = curr.left; }
        curr = stack.pop();
        res.add(curr.val);
        curr = curr.right;
    }
    return res;
}
```

### Under the hood
- A binary tree is just a graph with: one root, no cycles, and at most 2 children per node.
- BSTs aren't self-balancing by default. Inserting sorted input gives a degenerate "linked list" tree with O(n) operations.
- For balanced behavior, real systems use **AVL trees** (strict balance, fast lookup) or **Red-Black trees** (looser balance, faster modifications). Java's `TreeMap`/`TreeSet` use Red-Black.

### Gotchas & predict-the-output traps
- **Inorder of a BST yields sorted output.** This is the trick behind *Validate BST*, *Kth Smallest in BST*, and *Recover BST*.
- **`null` checks before recursing** — every tree problem hinges on the base case `if (root == null) return ...;`. Forgetting this is the #1 NPE source.
- **Returning the modified subtree.** Recursion patterns like `root.left = recurse(root.left, ...)` are easy to get wrong (forgetting the assignment leaves the tree unchanged).
- **A balanced *binary tree* is not the same as a *binary search tree*.** Don't confuse "balanced" (height) with "BST" (ordering).
- **Recursive depth on skewed trees can blow the stack.** Convert to iterative or use Morris traversal for O(1) space when interviewers push.

### Pattern template — Recursion shape (solve children, combine for parent)
```java
// Diameter of binary tree — return depth, track diameter as side effect
private int diameter = 0;
public int diameterOfBinaryTree(TreeNode root) {
    depth(root);
    return diameter;
}
private int depth(TreeNode n) {
    if (n == null) return 0;
    int l = depth(n.left), r = depth(n.right);
    diameter = Math.max(diameter, l + r);   // path through this node
    return 1 + Math.max(l, r);              // depth of subtree rooted here
}
```

### Expected DSA problem types
Massive interview category.
- **Traversals** — preorder, inorder, postorder, level-order (BFS)
- **Path problems** — *Path Sum, Binary Tree Maximum Path Sum, Diameter of Binary Tree*
- **Construction** — *Construct Binary Tree from Preorder and Inorder*
- **Lowest Common Ancestor** — *LCA of a Binary Tree, LCA of a BST*
- **BST validation** — *Validate Binary Search Tree*
- **Serialization** — *Serialize and Deserialize Binary Tree*
- **Views** — *Right Side View, Top View, Bottom View*
- **Recursion patterns** — almost every tree problem is "solve for the children, combine for the parent"

---

## 11. Trie (Prefix Tree)

### Etymology
**Coined in 1960 by Edward Fredkin** from the middle syllable of *re**trie**val*. He wanted it pronounced "tree," but everyone else says "try" to disambiguate from regular trees.

### Real-world example
**Phone keypad autocomplete** or **search bar suggestions**. Type "ja" and Java, JavaScript, Jakarta, January all surface. The structure is a tree where each node is one letter, and following a path spells a word.

### Problem it solves
Searching by **prefix**. A hash map can tell you if "java" is a stored word, but it can't tell you "give me all words starting with 'jav-'" without scanning every key.

### Complexity at a glance

| Operation              | Time   | Notes                              |
|------------------------|--------|------------------------------------|
| `insert(word)`         | O(L)   | L = length of word                 |
| `search(word)`         | O(L)   |                                    |
| `startsWith(prefix)`   | O(L)   |                                    |
| **Space**              | O(N·L·Σ) | N words × avg length × alphabet size; HashMap-based variant much smaller |

### Methods cheat sheet — full implementation
```java
class Trie {
    private static class Node {
        Node[] children = new Node[26];   // for lowercase a-z; use HashMap for general charset
        boolean isEnd;
    }
    private final Node root = new Node();

    // Insert a word
    public void insert(String word) {
        Node node = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (node.children[i] == null) node.children[i] = new Node();
            node = node.children[i];
        }
        node.isEnd = true;
    }

    // Exact match
    public boolean search(String word) {
        Node node = traverse(word);
        return node != null && node.isEnd;
    }

    // Prefix match
    public boolean startsWith(String prefix) {
        return traverse(prefix) != null;
    }

    private Node traverse(String s) {
        Node node = root;
        for (char c : s.toCharArray()) {
            int i = c - 'a';
            if (node.children[i] == null) return null;
            node = node.children[i];
        }
        return node;
    }
}
```

### Under the hood
- Each node represents one character position. The **root** is empty (no character).
- A boolean flag `isEnd` marks where a stored word terminates — needed so we can distinguish "car" stored vs "car" as a prefix of "card."
- Two common backing storages: fixed `Node[26]` (faster, more memory) or `Map<Character, Node>` (slower, much less memory for sparse alphabets like Unicode).

### Gotchas & predict-the-output traps
- **Forgetting `isEnd` makes prefix and word indistinguishable.** Inserting "card" without `isEnd=true` at the 'd' node means `search("card")` returns false even though every character exists.
- **Memory blows up with full 26-array nodes for sparse data.** A Trie of 1000 short words can take 100K+ nodes. Use HashMap-backed nodes when alphabet is large or input is sparse.
- **Case sensitivity** — `c - 'a'` only works for lowercase. Normalize input or use a HashMap node.
- **DFS in Word Search II must backtrack the visited mark** (set, then unset) — the same letter cell needs to be reusable for sibling branches.

### Pattern template — Word Search II (Trie + DFS on grid)
```java
public List<String> findWords(char[][] board, String[] words) {
    Trie trie = new Trie();
    for (String w : words) trie.insert(w);
    Set<String> result = new HashSet<>();
    int rows = board.length, cols = board[0].length;
    for (int r = 0; r < rows; r++)
        for (int c = 0; c < cols; c++)
            dfs(board, r, c, trie.root, new StringBuilder(), result);
    return new ArrayList<>(result);
}
// (Children navigation logic depends on the Node class — typical pattern is to walk `node.children[c-'a']`.)
```

### Expected DSA problem types
- **Implement Trie**
- **Word Search II** (Trie + DFS on grid — classic medium-hard)
- **Replace Words, Add and Search Word**
- **Autocomplete / typeahead**
- **Longest common prefix** of many strings

---

## 12. Graph

### Etymology
Greek *graphein* — "to write, to draw." Originally meant any drawing. In math, **Euler's 1736 paper on the Königsberg bridges** is considered the birth of graph theory — he drew the bridges and landmasses as dots and lines, calling it a *graph*.

### Real-world example
**A road map.** Cities are *nodes*; roads between them are *edges*. **Social networks** — people are nodes, friendships are edges. **The internet** — pages are nodes, hyperlinks are (directed) edges.

### Problem it solves
Anything where the entities matter *and* the connections between them matter. Trees are a special case of graphs (no cycles, one root).

### Complexity at a glance — V vertices, E edges

| Representation     | Space   | Add edge | Has edge? | List neighbors |
|--------------------|---------|----------|-----------|----------------|
| Adjacency list     | O(V+E)  | O(1)     | O(deg)    | O(deg)         |
| Adjacency matrix   | O(V²)   | O(1)     | O(1)      | O(V)           |

| Algorithm                | Time             | Notes                       |
|--------------------------|------------------|-----------------------------|
| BFS / DFS                | O(V+E)           | Each node + edge once       |
| Topological sort         | O(V+E)           | DFS or Kahn's               |
| Dijkstra (min-heap)      | O((V+E) log V)   | Non-negative edges only     |
| Bellman-Ford             | O(V·E)           | Handles negative edges      |
| Floyd-Warshall (all pairs)| O(V³)           | Dense, all-pairs            |
| Union-Find connectivity  | ~O(α(n)) per op  | α = inverse Ackermann ≈ 4   |

### Methods cheat sheet — building a graph
```java
import java.util.*;

// Adjacency list — most common; use this by default
Map<Integer, List<Integer>> graph = new HashMap<>();
void addEdge(int u, int v) {
    graph.computeIfAbsent(u, k -> new ArrayList<>()).add(v);
    graph.computeIfAbsent(v, k -> new ArrayList<>()).add(u);   // omit for directed
}

// Weighted edges — store [neighbor, weight] pairs
Map<Integer, List<int[]>> weighted = new HashMap<>();
void addWeighted(int u, int v, int w) {
    weighted.computeIfAbsent(u, k -> new ArrayList<>()).add(new int[]{v, w});
}

// Adjacency matrix — when V is small or graph is dense
int n = 10;
int[][] adj = new int[n][n];
adj[u][v] = 1;             // unweighted
adj[u][v] = weight;        // weighted (use a sentinel like Integer.MAX_VALUE for "no edge")

// From edge list (common LeetCode input format)
int[][] edges = {{0,1},{1,2},{2,0}};
Map<Integer, List<Integer>> g = new HashMap<>();
for (int[] e : edges) {
    g.computeIfAbsent(e[0], k -> new ArrayList<>()).add(e[1]);
    g.computeIfAbsent(e[1], k -> new ArrayList<>()).add(e[0]);
}
```

### Pattern template — DFS (recursive)
```java
Set<Integer> visited = new HashSet<>();
void dfs(int node, Map<Integer, List<Integer>> graph) {
    if (!visited.add(node)) return;       // add returns false if already present
    for (int nei : graph.getOrDefault(node, List.of())) {
        dfs(nei, graph);
    }
}
```

### Pattern template — DFS (iterative, to avoid stack overflow)
```java
void dfsIterative(int start, Map<Integer, List<Integer>> graph) {
    Deque<Integer> stack = new ArrayDeque<>();
    Set<Integer> visited = new HashSet<>();
    stack.push(start);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (!visited.add(node)) continue;
        for (int nei : graph.getOrDefault(node, List.of())) {
            if (!visited.contains(nei)) stack.push(nei);
        }
    }
}
```

### Pattern template — Topological sort (Kahn's algorithm, BFS-based)
```java
public int[] topoSort(int n, int[][] edges) {
    int[] indeg = new int[n];
    Map<Integer, List<Integer>> g = new HashMap<>();
    for (int[] e : edges) {
        g.computeIfAbsent(e[0], k -> new ArrayList<>()).add(e[1]);
        indeg[e[1]]++;
    }
    Queue<Integer> q = new ArrayDeque<>();
    for (int i = 0; i < n; i++) if (indeg[i] == 0) q.offer(i);
    int[] order = new int[n];
    int idx = 0;
    while (!q.isEmpty()) {
        int u = q.poll();
        order[idx++] = u;
        for (int v : g.getOrDefault(u, List.of())) {
            if (--indeg[v] == 0) q.offer(v);
        }
    }
    return idx == n ? order : new int[0];   // empty if cycle exists
}
```

### Pattern template — Dijkstra's shortest path
```java
public int[] dijkstra(int n, Map<Integer, List<int[]>> graph, int src) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));
    pq.offer(new int[]{src, 0});
    while (!pq.isEmpty()) {
        int[] cur = pq.poll();
        int u = cur[0], d = cur[1];
        if (d > dist[u]) continue;          // stale entry, skip
        for (int[] edge : graph.getOrDefault(u, List.of())) {
            int v = edge[0], w = edge[1];
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.offer(new int[]{v, dist[v]});
            }
        }
    }
    return dist;
}
```

### Gotchas & predict-the-output traps
- **Always check `visited` BEFORE adding to the queue/stack**, not just on dequeue. Otherwise the same node can be enqueued many times, blowing memory and breaking BFS distance correctness.
- **`HashSet.add` returns `boolean`** — use `if (visited.add(x))` as the idiomatic "check-and-mark."
- **Undirected graphs need both directions inserted** in the adjacency list. Forgetting one direction is a silent bug.
- **DFS recursion depth** on a chain of 10K nodes blows the JVM stack (~10K frames). Convert to iterative when N is large.
- **Dijkstra fails on negative edges.** Use Bellman-Ford if negatives are possible.
- **Topological sort only exists for DAGs.** If a cycle is present, Kahn's algorithm produces an incomplete ordering — check `idx == n` to detect.

### Expected DSA problem types
- **BFS** — *Number of Islands, Rotting Oranges, Word Ladder, Open the Lock*
- **DFS** — *Number of Islands (DFS variant), Surrounded Regions, Pacific Atlantic Water Flow*
- **Topological sort** — *Course Schedule I & II, Alien Dictionary*
- **Shortest path (Dijkstra)** — *Network Delay Time, Cheapest Flights Within K Stops*
- **Cycle detection** — both directed (DFS coloring) and undirected (Union-Find)
- **Connected components** — *Number of Provinces, Number of Connected Components in an Undirected Graph*

---

## 13. Union-Find (Disjoint Set Union)

### Etymology
Descriptive — a structure for *finding* which set something belongs to and *uniting* sets together. Sometimes called **DSU** for Disjoint Set Union.

### Real-world example
**Merging companies.** Each company has employees. When two companies merge, all their employees now belong to the same parent organization. Given any two employees, you can ask: "Do they work for the same parent company now?"

### Problem it solves
"Given a stream of merge operations and connectivity queries, answer them all fast." Naive approach is BFS each time — O(V+E) per query. Union-Find is **near-O(1) per query** with path compression + union by rank.

### Complexity at a glance

| Operation                    | Time           | Notes                              |
|------------------------------|----------------|------------------------------------|
| `find(x)`                    | ~O(α(n)) ≈ O(1) | With path compression             |
| `union(a, b)`                | ~O(α(n)) ≈ O(1) | With union by rank/size           |
| Init                         | O(n)           |                                    |
| **Space**                    | O(n)           | parent[] + rank[]                  |

`α(n)` is the inverse Ackermann function. For any practical n, α(n) ≤ 4. So treat ops as O(1).

### Methods cheat sheet — full implementation
```java
class UnionFind {
    private final int[] parent;
    private final int[] rank;     // upper bound on tree height
    private int components;       // optional: tracks number of disjoint sets

    public UnionFind(int n) {
        parent = new int[n];
        rank   = new int[n];
        components = n;
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    // Find with PATH COMPRESSION — flattens the tree on the way up
    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    // Iterative path compression (avoids recursion depth issues)
    public int findIter(int x) {
        int root = x;
        while (parent[root] != root) root = parent[root];
        while (parent[x] != root) {
            int next = parent[x];
            parent[x] = root;
            x = next;
        }
        return root;
    }

    // Union by RANK — attach shorter tree under taller
    public boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;          // already connected
        if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
        parent[rb] = ra;
        if (rank[ra] == rank[rb]) rank[ra]++;
        components--;
        return true;
    }

    public boolean connected(int a, int b) { return find(a) == find(b); }
    public int components() { return components; }
}
```

### Under the hood
- Each set is represented by a tree of parent pointers; the root is the set's representative.
- **Path compression**: during `find`, every node on the path is reattached directly to the root, so subsequent finds on the same path are O(1).
- **Union by rank** (or size): always attach the shorter tree under the taller one to keep heights small.
- Together, these two optimizations give the famous α(n) amortized cost.

### Gotchas & predict-the-output traps
- **Recursive `find` blows the stack** on adversarial input (a long chain before any compression). Use iterative `findIter` for safety on large inputs.
- **Compare ROOTS, not raw values.** `connected(a, b)` must be `find(a) == find(b)`, not `parent[a] == parent[b]`.
- **Order of compression matters.** Compress BEFORE comparing roots in union: read both roots first, then assign — otherwise the swap-by-rank logic uses stale parents.
- **Without path compression OR union by rank**, worst case degenerates to O(n) per find. Always include at least one optimization (both is standard).

### Pattern template — Counting connected components
```java
public int countComponents(int n, int[][] edges) {
    UnionFind uf = new UnionFind(n);
    for (int[] e : edges) uf.union(e[0], e[1]);
    return uf.components();
}
```

### Expected DSA problem types
- **Connectivity / components** — *Number of Provinces, Number of Connected Components*
- **Cycle detection in undirected graph** — *Graph Valid Tree, Redundant Connection*
- **Kruskal's MST**
- **Account merging** — *Accounts Merge*
- **Dynamic connectivity** problems where edges are added incrementally

---

## 14. Strings (Java's Quirks)

### Etymology
*String* is Old English *streng* — "a cord, a line." Came to mean "a sequence" — a string of pearls, a string of letters.

### Why strings deserve their own section in Java
- **Immutable.** Every `+`, `replace`, or `substring` creates a new object. In a loop, this is O(n²).
- **Use `StringBuilder`** for concatenation in loops — O(n) total.
- **`String.equals()`** for content comparison; `==` checks reference identity.
- **`charAt(i)`** is O(1). `substring(i, j)` in Java 7+ creates a copy — O(n).

### Complexity at a glance

| Operation                          | Time   | Notes                                |
|------------------------------------|--------|--------------------------------------|
| `length()`                         | O(1)   | Stored as a field                    |
| `charAt(i)`                        | O(1)   |                                      |
| `equals` / `compareTo`             | O(n)   |                                      |
| `substring(i, j)`                  | O(n)   | Java 7+ — copies                     |
| `+` concatenation                  | O(m+n) | New object each time                 |
| `s += x` in a loop                 | O(n²)  | Quadratic — use StringBuilder        |
| `StringBuilder.append`             | Amortized O(1) |                              |
| `indexOf` (substring)              | O(n·m) | Naive scan; not KMP                  |

### Methods cheat sheet
```java
String s = "Hello, World";

// Length, indexing, slicing
int len = s.length();
char c  = s.charAt(0);                // 'H'
String sub = s.substring(7);          // "World"
String sub2= s.substring(7, 12);      // "World" — [start, end)

// Searching
boolean has = s.contains("World");
int idx     = s.indexOf("o");          // 4 (first)
int last    = s.lastIndexOf("o");      // 8
int from    = s.indexOf("o", 5);       // 8 (search from index 5)

// Comparison
boolean same = s.equals("Hello, World");
boolean ci   = s.equalsIgnoreCase("HELLO, WORLD");
int cmp      = s.compareTo("abc");     // < 0, 0, > 0
boolean pre  = s.startsWith("Hello");
boolean suf  = s.endsWith("World");

// Transformation (each returns a NEW string)
String up = s.toUpperCase();
String lo = s.toLowerCase();
String tr = s.trim();                   // strips ASCII whitespace
String st = s.strip();                  // Java 11+: Unicode-aware
String rp = s.replace('o', '0');        // char → char
String rs = s.replace("World", "Java"); // CharSequence → CharSequence (NOT regex)
String rg = s.replaceAll("[aeiou]", "*"); // REGEX
String rf = s.replaceFirst("[aeiou]", "*");

// Splitting / joining
String[] parts = s.split(", ");
String csv = String.join(",", "a", "b", "c");        // "a,b,c"
String csv2= String.join("-", List.of("x","y","z")); // "x-y-z"

// Convert
char[] chars = s.toCharArray();
String back  = new String(chars);
String iStr  = String.valueOf(42);
String fmt   = String.format("x=%d, y=%.2f", 10, 3.14159);

// CHAR ↔ INT TRICKS — common in DSA
int digit = '7' - '0';            // 7   (digit char to int)
int idx26 = 'c' - 'a';            // 2   (lowercase letter to 0-25)
char back2 = (char) ('a' + 2);    // 'c'
boolean isLetter = Character.isLetter(c);
boolean isDigit  = Character.isDigit(c);
boolean isAlnum  = Character.isLetterOrDigit(c);
char lower = Character.toLowerCase(c);

// STRINGBUILDER — mutable, USE THIS in loops
StringBuilder sb = new StringBuilder();
sb.append("hi").append(' ').append(42);    // chainable
sb.insert(0, "START: ");
sb.deleteCharAt(0);
sb.delete(0, 3);                            // [start, end)
sb.reverse();
sb.setCharAt(2, 'Z');
char ch = sb.charAt(0);
int sblen = sb.length();
sb.setLength(0);                            // empty without reallocating
String done = sb.toString();
```

### Under the hood
- `String` is backed by a `byte[]` (Java 9+ "compact strings" — Latin-1 if all chars fit, UTF-16 otherwise).
- The string pool: literal strings (`"hello"`) are deduplicated in a JVM-managed pool. `new String("hello")` bypasses the pool. `s.intern()` adds to the pool. This is why `==` sometimes "works" on strings — purely a coincidence of pooling.
- `StringBuilder` (not synchronized, fast) vs `StringBuffer` (synchronized, slow). Use `StringBuilder`.

### Gotchas & predict-the-output traps
- **`==` vs `equals`:**
  ```java
  String a = "hi";
  String b = "hi";
  String c = new String("hi");
  a == b;              // true — same pooled literal
  a == c;              // false — c is a fresh object
  a.equals(c);         // true — content match
  ```
- **`s += "x"` inside a loop is O(n²).** Always reach for `StringBuilder` in any loop that builds a string.
- **`replace` is NOT regex; `replaceAll` IS regex.** Calling `replaceAll(".", "x")` replaces every character because `.` is the regex wildcard.
- **`split` is regex-based.** Splitting on `.` (literal dot) requires `s.split("\\.")`.
- **`substring` in Java 7+ copies** — earlier JDK versions shared the backing array, which caused memory leaks when small substrings kept large strings alive. The modern copy is safer but O(n).
- **`Character.getNumericValue('A')` returns 10**, not the codepoint — surprising for non-decimal characters.
- **`String.format` uses platform locale** by default. `%f` in some locales uses comma as decimal separator. Use `String.format(Locale.ROOT, ...)` for predictable output.

### Pattern template — Frequency comparison (anagram check)
```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] count = new int[26];
    for (int i = 0; i < s.length(); i++) {
        count[s.charAt(i) - 'a']++;
        count[t.charAt(i) - 'a']--;
    }
    for (int v : count) if (v != 0) return false;
    return true;
}
```

### Pattern template — Sliding window with char counts (longest substring without repeating)
```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int best = 0, left = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1;
        }
        lastSeen.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

### Expected DSA problem types
- **Palindrome problems** — *Valid Palindrome, Longest Palindromic Substring*
- **Anagrams** — *Group Anagrams, Valid Anagram*
- **Pattern matching** — *Implement strStr (KMP), Repeated String Pattern*
- **String DP** — *Edit Distance, Longest Common Subsequence*
- **Sliding window over chars** — *Longest Substring Without Repeating Characters, Minimum Window Substring*

---

## 15. Advanced (Brief Mention)

These show up at the harder end and are **not Karat-priority** but worth knowing they exist.

### Segment Tree
Range queries (sum, min, max) with point updates in O(log n). Used in *Range Sum Query - Mutable*. Java has no built-in; you write an array-based version.

### Fenwick Tree (Binary Indexed Tree, BIT)
Lighter alternative to segment tree for prefix sums with updates. O(log n) for both. Slick bit-manipulation implementation.

### Suffix Array / Suffix Tree
Advanced string structures. Mostly competitive programming territory.

### Skip List
Probabilistic alternative to balanced BSTs. Used in `ConcurrentSkipListMap`.

---

## Cheat Sheet for Karat-Style Interviews

| Pattern you see in the problem | Reach for |
|---|---|
| "Find pair / triple summing to X" | HashMap |
| "Top K / Kth largest" | PriorityQueue |
| "Shortest path in unweighted graph" | Queue (BFS) |
| "Levels of a tree" | Queue (BFS) |
| "All paths in a tree" | Recursion / Stack (DFS) |
| "Balanced parens / next greater" | Stack |
| "Sliding window max" | Deque |
| "Range / sorted lookups" | TreeMap |
| "Prefix lookup" | Trie |
| "Connected components / dynamic merge" | Union-Find |
| "Insertion order matters with O(1) ops" | LinkedHashMap |
| "Merge K sorted things" | PriorityQueue |
| "Cycle in linked list / find middle" | Two pointers |

---

## Composite Cheat Sheet — Cross-Structure Comparisons

When you're choosing between two similar things, this is the table:

### List implementations
| Need | Choose | Why |
|---|---|---|
| Random access by index, growing | `ArrayList` | O(1) get, cache-friendly |
| Heavy add/remove at both ends | `ArrayDeque` (as Deque) | O(1) ends, no node alloc |
| `List` interface + Deque ops, allow nulls | `LinkedList` | Implements all three |
| Thread-safe, mostly reads | `CopyOnWriteArrayList` | Reads lock-free |

### Map implementations
| Need | Choose | Why |
|---|---|---|
| Fastest get/put, no ordering | `HashMap` | O(1) average |
| Sorted by key, range queries | `TreeMap` | O(log n), navigable |
| Insertion-order iteration | `LinkedHashMap` (default mode) | O(1) + ordered |
| Recently-used eviction | `LinkedHashMap` (access mode) | Built-in LRU |
| Thread-safe, high concurrency | `ConcurrentHashMap` | Lock striping |

### Set implementations — same axes as Map
| Need | Choose |
|---|---|
| Fast membership, no order | `HashSet` |
| Sorted, range queries | `TreeSet` |
| Insertion-order iteration | `LinkedHashSet` |
| Concurrent | `ConcurrentSkipListSet` / wrap with `Collections.synchronizedSet` |

### Stack/Queue implementations
| Need | Choose | Avoid |
|---|---|---|
| Stack semantics | `ArrayDeque` | `java.util.Stack` (legacy, synchronized) |
| FIFO queue | `ArrayDeque` | `LinkedList` (slower) |
| Priority order | `PriorityQueue` | — |
| Thread-safe queue | `ConcurrentLinkedQueue`, `ArrayBlockingQueue` | — |

---

## How to Internalize This

The structures aren't independent — they compose:
- **LRU Cache** = HashMap + Doubly Linked List
- **Min Stack** = Stack + Stack (or Stack of pairs)
- **Top K Frequent** = HashMap (count) + PriorityQueue (rank)
- **Word Search II** = Trie + DFS + visited set
- **Dijkstra** = Graph + PriorityQueue + HashMap (distances)

When you see a problem, the question isn't "which one structure" — it's "which combination gives me the right operations cheaply." That mental model is what separates fluent DSA from memorized DSA.

---

## Predict-the-Output Practice (Karat Part 1 Style)

Cover the explanation, predict the output, then check. These are the kinds of subtleties that show up in tricky-Java rounds.

---

**Q1.**
```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4, 5));
list.remove(2);
System.out.println(list);
```
<details><summary>Answer</summary>

`[1, 2, 4, 5]`

`remove(2)` resolves to `remove(int index)` because `2` is a primitive `int`. It removes the element AT index 2, which is `3`. To remove the *value* 2, you'd write `list.remove(Integer.valueOf(2))`.
</details>

---

**Q2.**
```java
Map<String, Integer> m = new HashMap<>();
m.put("a", null);
int x = m.get("a");
System.out.println(x);
```
<details><summary>Answer</summary>

**Throws `NullPointerException`** at the assignment line. `m.get("a")` returns `Integer` of value `null`. Auto-unboxing to `int` calls `.intValue()` on `null`. Use `getOrDefault` or check for null.
</details>

---

**Q3.**
```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(3); pq.offer(1); pq.offer(2);
StringBuilder sb = new StringBuilder();
for (int x : pq) sb.append(x).append(' ');
System.out.println(sb.toString().trim());
```
<details><summary>Answer</summary>

`1 3 2` (or similar — NOT `1 2 3`)

Iterating a `PriorityQueue` walks the underlying heap array, NOT in sorted order. Only `poll()` returns elements in priority order. The exact output depends on heap layout but it's NOT sorted.
</details>

---

**Q4.**
```java
String a = "hello";
String b = "hello";
String c = new String("hello");
System.out.println((a == b) + " " + (a == c) + " " + a.equals(c));
```
<details><summary>Answer</summary>

`true false true`

`a` and `b` both refer to the same pooled string literal. `c` is constructed with `new`, so it's a fresh object on the heap — `a == c` is false. Content equality (`equals`) is true.
</details>

---

**Q5.**
```java
Deque<Integer> d = new ArrayDeque<>();
d.offer(1); d.offer(2); d.offer(3);
d.push(99);
System.out.println(d.poll());
```
<details><summary>Answer</summary>

`99`

`offer` adds to the tail (queue semantics). `push` adds to the HEAD (deque/stack semantics). `poll` removes from the head. So after offers the deque is `[1, 2, 3]`; after `push(99)` it's `[99, 1, 2, 3]`; `poll()` returns `99`.
</details>

---

**Q6.**
```java
Map<Integer, Integer> m = new HashMap<>();
for (int i = 0; i < 5; i++) m.put(i, i * 10);
m.merge(2, 5, Integer::sum);
m.merge(99, 7, Integer::sum);
System.out.println(m.get(2) + " " + m.get(99));
```
<details><summary>Answer</summary>

`25 7`

`merge(k, v, fn)`: if key absent, puts `v`. If present, replaces with `fn(oldValue, v)`. So key 2 (value 20) becomes `20 + 5 = 25`. Key 99 didn't exist, so it just gets the value 7.
</details>

---

**Q7.**
```java
TreeMap<String, Integer> tm = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
tm.put("hello", 1);
tm.put("HELLO", 2);
tm.put("Hello", 3);
System.out.println(tm.size() + " " + tm.get("hello"));
```
<details><summary>Answer</summary>

`1 3`

`TreeMap` uses the comparator (case-insensitive here) to determine key identity, NOT `equals`. All three puts target the same logical key. The last one wins, so the value is 3 and size is 1.
</details>

---

**Q8.**
```java
String s = "abc";
StringBuilder sb = new StringBuilder("abc");
System.out.println(s.equals(sb));
System.out.println(s.contentEquals(sb));
```
<details><summary>Answer</summary>

`false`
`true`

`String.equals(Object)` returns false unless the argument is a `String`. `StringBuilder` is not a `String`, so it's false. `String.contentEquals(CharSequence)` checks the actual character sequence regardless of type — true.
</details>

---

**Q9.**
```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4));
for (Integer n : list) {
    if (n == 2) list.remove(n);
}
System.out.println(list);
```
<details><summary>Answer</summary>

**Throws `ConcurrentModificationException`** (in most cases — JDK behavior varies slightly).

Modifying a list inside a `for-each` loop invalidates the iterator. The fix is `list.removeIf(n -> n == 2)` or using an explicit `Iterator` and calling `it.remove()`.
</details>

---

**Q10.**
```java
int[] arr = {3, 1, 4, 1, 5};
List<int[]> wrapped = Arrays.asList(arr);
System.out.println(wrapped.size());
```
<details><summary>Answer</summary>

`1`

`Arrays.asList` for a primitive array returns a single-element list containing the array itself (`List<int[]>` of size 1). To get a `List<Integer>`, use `Arrays.stream(arr).boxed().toList()`.
</details>

---

**Q11.**
```java
HashSet<int[]> set = new HashSet<>();
int[] a = {1, 2, 3};
set.add(a);
a[0] = 99;
System.out.println(set.contains(a));
```
<details><summary>Answer</summary>

`true` — but for a misleading reason.

Arrays use `Object`'s default `hashCode` (identity-based), so the hash doesn't change when contents change. `contains` finds it because we're looking up the same object. But `set.contains(new int[]{99, 2, 3})` would return false even though the contents match. **Never use raw arrays as set/map keys** — use `List<Integer>` or wrap them.
</details>

---

**Q12.**
```java
PriorityQueue<Integer> pq = new PriorityQueue<>((a, b) -> b - a);
pq.offer(Integer.MIN_VALUE);
pq.offer(2);
System.out.println(pq.peek());
```
<details><summary>Answer</summary>

`Integer.MIN_VALUE` (i.e., `-2147483648`) — the **opposite** of what you intended (max-heap should give `2`).

`b - a` overflows: `2 - Integer.MIN_VALUE` is mathematically `2147483650`, but in `int` it wraps to a negative number. The comparator lies, the heap reports the wrong root. Use `Integer.compare(b, a)` instead — it's overflow-safe.
</details>

---

**Q13.**
```java
LinkedHashMap<Integer, String> lru = new LinkedHashMap<>(16, 0.75f, true);
lru.put(1, "a"); lru.put(2, "b"); lru.put(3, "c");
lru.get(1);
lru.put(4, "d");
System.out.println(lru.keySet());
```
<details><summary>Answer</summary>

`[2, 3, 1, 4]`

Access-order mode moves accessed entries to the tail. Initial order after puts: `[1, 2, 3]`. After `get(1)`: `[2, 3, 1]`. After `put(4, "d")`: `[2, 3, 1, 4]`. (Without overriding `removeEldestEntry`, no eviction happens — the map just grows.)
</details>

---

**Q14.**
```java
Map<Integer, Integer> m = new HashMap<>();
m.put(null, 1);
m.put(null, 2);
System.out.println(m.size() + " " + m.get(null));
```
<details><summary>Answer</summary>

`1 2`

`HashMap` allows one `null` key. The second `put(null, 2)` overwrites the first. (`Hashtable` and `ConcurrentHashMap` would throw NPE — common interview discriminator.)
</details>

---

**Q15.**
```java
String s = "ababab";
String[] parts = s.split("a");
System.out.println(parts.length + " | " + Arrays.toString(parts));
```
<details><summary>Answer</summary>

`4 | [, b, b, b]`

`split("a")` gives `["", "b", "b", "b"]`. The empty string at index 0 is the part before the first `a`. By default, `split` removes *trailing* empty strings but keeps leading and middle ones. `s.split("a", -1)` would also keep trailing empties.
</details>

---

## Closing Note — Building Mental Muscle

The goal of these notes isn't to memorize methods. It's to internalize the **shape** of each structure so that when you read a problem statement, the right tool surfaces unprompted.

Three habits that accelerate this:

1. **Re-derive, don't re-read.** Cover the methods cheat sheet and write out the operations you remember. Compare. Repeat in 24 hours, then 3 days, then a week.
2. **Hand-implement at least once.** You'll get more from writing your own `LinkedList`, `Trie`, and `UnionFind` from scratch than from reading any documentation. Type the code; don't paste.
3. **Solve, then re-solve in a different structure.** *Two Sum* with HashMap, then with sorted array + two pointers. *LRU* with `LinkedHashMap`, then hand-rolled. The act of switching forces you to articulate the trade-off explicitly.

When the trade-offs feel obvious — "this needs sorted ordering, so HashMap is out" — you've moved from memorization to fluency. That's the bar.
