# 05 — Strings & Immutability

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## 📚 Study Material

### 1. String Immutability — Why It Matters

A `String` object **cannot be changed after creation**. Every "modification" creates a new String.

```java
String s = "hello";
s.toUpperCase();      // returns "HELLO" — s is STILL "hello"
s = s.toUpperCase();  // now s points to new "HELLO" object — old "hello" is eligible for GC
```

**Why immutable? (memorise these five reasons)**

| Reason | Explanation |
|--------|-------------|
| **Thread safety** | Shared freely between threads — no synchronisation needed |
| **Security** | File paths, class names, network URLs can't be tampered with after creation |
| **String pool** | Only works because strings can't change — sharing is safe |
| **hashCode caching** | Computed once, cached for life — `HashMap` key lookups are O(1) |
| **Class design** | `String` is `final` — can't be subclassed to break immutability |

```java
// hashCode caching — String computes it lazily on first call, then caches
public int hashCode() {
    int h = hash;              // cached value (0 means not computed yet)
    if (h == 0 && !hashIsZero) {
        h = /* compute from chars */;
        hash = h;              // cached — never recomputed
    }
    return h;
}
```

### 2. String Pool (Intern Pool)

```
   HEAP MEMORY
   ┌─────────────────────────────────────┐
   │                                     │
   │  String Pool (inside heap since J7) │
   │  ┌─────────────────────────────┐    │
   │  │  "hello"  "world"  "Java"  │    │
   │  └─────────────────────────────┘    │
   │                                     │
   │  Regular heap objects               │
   │  ┌──────┐  ┌──────┐                │
   │  │"hello"│  │"hello"│  ← new String("hello") creates here  │
   │  └──────┘  └──────┘                │
   └─────────────────────────────────────┘
```

```java
String a = "hello";                 // literal → pool (or reuses existing)
String b = "hello";                 // same literal → same pool reference
String c = new String("hello");     // explicit new → new object on heap (NOT in pool)
String d = c.intern();              // forces c's value into pool, returns pool reference

a == b;      // true  — same pool object
a == c;      // false — c is on heap, a is in pool
a == d;      // true  — intern() returned the pool reference
a.equals(c); // true  — same characters

// SINCE JAVA 7: Pool is in the regular heap (was in PermGen before)
// This means: pool strings can be garbage collected when unreferenced
```

💡 **Rule:** Always use `equals()` for string comparison. Never rely on `==` (it checks reference, not value).

### 3. StringBuilder vs StringBuffer vs String Concatenation

```java
// PROBLEM: String concatenation in a loop is O(n²)
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;   // creates a NEW String every iteration
    // Iteration 1: copy 1 char,  create new String
    // Iteration 2: copy ~2 chars, create new String
    // ...
    // Iteration N: copy ~N chars → total copies ≈ N²/2
}

// SOLUTION: StringBuilder — single mutable buffer, O(n) total
StringBuilder sb = new StringBuilder(16);  // initial capacity (avoids early resizes)
for (int i = 0; i < 10000; i++) {
    sb.append(i);   // appends to internal char[] — resizes by doubling when needed
}
String result = sb.toString();             // single String creation at the end
```

| Type | Mutable | Thread-safe | Performance | When to use |
|------|:-------:|:-----------:|:-----------:|-------------|
| `String` | ❌ | ✅ (immutable) | Slow for building | Single values, constants |
| `StringBuilder` | ✅ | ❌ | Fast | Loops, building strings (99% of cases) |
| `StringBuffer` | ✅ | ✅ (synchronized) | Slower | Legacy — avoid in new code |

```java
// Compiler optimisation: single-expression concat is fine
String full = first + " " + last;  // compiler converts to StringBuilder (or invokedynamic in Java 9+)
// No need to manually use StringBuilder for one-liners
```

### 4. String Methods — Key Interview Ones

```java
String s = "  Hello, World!  ";

s.trim();                    // "Hello, World!" — removes ASCII whitespace from both ends
s.strip();                   // same, but handles Unicode whitespace (Java 11)
s.stripLeading();            // "Hello, World!  " (Java 11)
s.stripTrailing();           // "  Hello, World!" (Java 11)

s.substring(2, 7);           // "Hello" — start inclusive, end EXCLUSIVE
s.indexOf("World");          // 9
s.contains("Hello");         // true
s.startsWith("  He");        // true
s.replace("World", "Java");  // "  Hello, Java!  " — replaces ALL occurrences
s.split(", ");               // ["  Hello", "World!  "]
s.toCharArray();             // char[] — useful for character-level processing

// Java 11+
"  ".isBlank();              // true — empty or only whitespace
"hello\nworld".lines();      // Stream<String> — ["hello", "world"]
"ha".repeat(3);              // "hahaha"

// Java 15+: Text blocks
String json = """
        {
            "name": "Alice",
            "role": "trader"
        }
        """;
// Strips common leading whitespace, preserves relative indentation
```

### 5. char[] for Sensitive Data

```java
// Strings stay in memory until GC (and in the pool potentially forever)
// char[] can be explicitly zeroed after use

char[] password = getPassword();
try {
    authenticate(password);
} finally {
    Arrays.fill(password, '\0');  // explicit wipe — data gone from memory
}

// This is why java.io.Console.readPassword() returns char[], not String
// OWASP recommends char[] for passwords, PINs, tokens, etc.
```

### 6. How to Create an Immutable Class (General Pattern)

```java
public final class Money {                        // 1. final class — can't subclass
    private final BigDecimal amount;               // 2. final fields — can't reassign
    private final Currency currency;
    private final List<String> tags;

    public Money(BigDecimal amount, Currency currency, List<String> tags) {
        this.amount = amount;                      // 3. values via constructor only
        this.currency = currency;
        this.tags = List.copyOf(tags);             // 4. defensive copy of mutable input
    }

    public BigDecimal getAmount() { return amount; }      // immutable — safe to return
    public Currency getCurrency() { return currency; }    // immutable — safe to return
    public List<String> getTags() { return tags; }        // 5. already immutable copy — safe
    // NO setters                                          // 6. no mutators
    
    // equals, hashCode, toString — implement based on all fields
}
```

**Five rules:** (1) `final` class, (2) `private final` fields, (3) constructor only, (4) defensive copy mutable inputs, (5) defensive copy mutable outputs (or return immutable versions).

---

## Rapid-Fire Q&A

### Q1: Why is `String` immutable?
**A:** Security (can't modify file paths, class names), thread safety (shared without synchronisation), String pool works because interned strings can't change, `hashCode` can be cached (used as HashMap keys). `String` is `final` — can't subclass.

### Q2: What's the String pool?
**A:** JVM maintains a pool of unique string literals in the heap (since Java 7 — was in PermGen before). `"hello"` and `"hello"` point to the same object. `new String("hello")` creates a separate object on the heap (not in pool). `intern()` forces a string into the pool.

### Q3: `==` vs `equals()` for Strings?
**A:** `==` compares references. `equals()` compares characters. `"hello" == "hello"` → true (same pool object). `new String("hello") == "hello"` → false (different objects). Always use `equals()`.

### Q4: `String` vs `StringBuilder` vs `StringBuffer`?
**A:** `String`: immutable — concatenation creates new objects. `StringBuilder`: mutable, not thread-safe, fast. `StringBuffer`: mutable, thread-safe (synchronized), slower. Use `StringBuilder` for loops/building. `+` in a single expression is optimised by the compiler.

### Q5: Why not concatenate strings in a loop?
**A:** Each `+=` creates a new `String` object. N iterations = N intermediate strings = O(n²) copying. `StringBuilder.append()` is O(n) total — single buffer.

### Q6: What does `String.intern()` do?
**A:** Adds the string to the pool (or returns the existing pooled instance). After `intern()`, you can use `==` for comparison. Rarely needed in application code — mainly for memory optimisation in frameworks.

### Q7: What's immutability in general? How do you make a class immutable?
**A:** Object state can't change after construction. Rules: `final` class, `private final` fields, no setters, values only via constructor, defensive copy mutable fields (input AND output). Java 16 `record` gives most of this for free.

### Q8: `char[]` vs `String` for passwords?
**A:** `String` stays in memory until GC collects it (and in the string pool potentially forever). `char[]` can be explicitly zeroed after use (`Arrays.fill(chars, '0')`). Security-sensitive — OWASP recommends `char[]`.

---

## Can you answer these cold?

- [ ] String immutability — why it matters (security, thread safety, pool, hashCode caching)
- [ ] String pool — where it lives, `intern()`, `new String()` vs literal
- [ ] `StringBuilder` vs `StringBuffer` — when each
- [ ] Loop concatenation performance — O(n²) problem
- [ ] How to create an immutable class — 5 rules

[← Back to Index](./00_INDEX.md)
