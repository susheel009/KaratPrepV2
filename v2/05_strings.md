# 05 — Strings & Immutability

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/05_strings_doubts.md) · [← Prev: 04 Exception Handling](./04_exceptions.md) · [Next: 06 Java 8+ →](./06_java8_plus.md)
>
> **Priority:** 🟡 High · **Related topics:** [03 OOP](./03_oop_language.md) · [06 Java 8+ (text blocks)](./06_java8_plus.md) · [07 JVM (String pool)](./07_jvm.md) · [02 Collections (hashCode in keys)](./02_collections.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — printed signs and shared stamps
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — pool vs heap, `==` vs `.equals`
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — minimal demo
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Immutability](#41-immutability--what-and-why)
   - [4.2 == vs .equals](#42--vs-equals--reference-vs-content)
   - [4.3 The String pool](#43-the-string-pool--shared-literals-and-intern)
   - [4.4 StringBuilder vs StringBuffer](#44-stringbuilder-vs-stringbuffer--mutable-buffers)
   - [4.5 Common String methods](#45-common-string-methods-the-ones-you-need-cold)
   - [4.6 Text blocks](#46-text-blocks-java-15)
   - [4.7 char[] for sensitive data](#47-char-for-sensitive-data)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 Compact strings](#51-compact-strings-java-9-bytes--coder)
   - [5.2 hashCode caching](#52-hashcode-caching)
   - [5.3 Concat compilation](#53-string-concat-compilation-java-9-invokedynamic)
   - [5.4 intern() and GC](#54-intern-mechanics-and-gc-implications)
   - [5.5 Unicode and surrogate pairs](#55-unicode-surrogates-and-codepoints)
   - [5.6 substring() in Java 7+](#56-substring-after-java-7-no-more-shared-array)
   - [5.7 Building an immutable class](#57-the-five-rules-for-an-immutable-class)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Picture a small village post office in 1955.

On the front wall hangs a **printed metal sign**: `OPEN 8AM – 5PM, MONDAY–FRIDAY`. The sign was made at the foundry. To "change" it — say, to extend Saturday hours — they don't get out a chisel and edit the metal. They send the sign to the foundry to be **destroyed**, and a new sign is cast and shipped back. The old sign is gone; the new one takes its place on the wall.

That's **immutability**. A `String` in Java is exactly that metal sign. You can't edit one — you can only swap one for another.

Inside the post office, the postmaster has a **drawer of rubber stamps**: one for `"PAID"`, one for `"RECEIVED"`, one for `"REGISTERED"`. When two clerks need the `"PAID"` stamp, they don't carve a new stamp — they share the one in the drawer. If a clerk needs a stamp that doesn't exist yet, the drawer gets a new one made and slots it in. Once a stamp is in the drawer, anyone in the office can grab the same physical stamp.

That's the **String pool**. Java keeps a drawer of every `String` literal in your code. Two `"hello"`s in two different methods aren't two strings — they're the same single shared string in the pool. New `String("hello")` is the postmaster carving a brand-new "PAID" stamp on the side and keeping it on his own desk, ignoring the drawer entirely.

These two ideas — **the sign that can never be edited** and **the drawer of shared stamps** — explain almost every weird String behaviour in Java.

> Once you see Strings as printed signs and the pool as a drawer of shared stamps, every gotcha in this file (`==` vs `.equals`, `intern()`, why `+=` in a loop is slow, why `char[]` is preferred for passwords) is just a consequence.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

Trace these five lines, one at a time, watching what's in the pool and what's on the heap:

```
String s1 = "hello";                    // [1]
String s2 = "hello";                    // [2]
String s3 = new String("hello");        // [3]
boolean a = (s1 == s2);                 // [4]
boolean b = (s1 == s3);                 // [5]
boolean c = s1.equals(s3);              // [6]
```

| Line | What happens | Pool | Heap (separate from pool) |
|:--:|---|---|---|
| 1 | The compiler sees the literal `"hello"`. It either reuses the pooled `"hello"` if present, or adds it. `s1` points to the pool entry. | `["hello"]` | (empty) |
| 2 | Another `"hello"` literal. The pool already has one — reuse it. `s2` points to the **same** object as `s1`. | `["hello"]` | (empty) |
| 3 | `new String("hello")` **forces a brand-new** `String` object on the regular heap. The compiler hands the pooled `"hello"` to the `String(String)` constructor as the source data. `s3` now points to the new heap object — *not* the pool one. | `["hello"]` | `["hello" — heap]` |
| 4 | `s1 == s2` compares **references**. Both reference the pool's `"hello"`. **`true`.** | | |
| 5 | `s1 == s3` compares references. `s1` is the pool entry, `s3` is the heap entry — different objects. **`false`.** | | |
| 6 | `s1.equals(s3)` compares **contents** (character by character). Both are `"hello"`. **`true`.** | | |

Two takeaways the walkthrough makes obvious:

1. **`==` on Strings asks "same object?". `.equals` asks "same characters?".** Almost always you want `.equals`.
2. **`new String("...")` is almost always wrong.** It deliberately bypasses the pool and creates a duplicate. Use the literal directly.

Now the actual code, plus the `StringBuilder` performance fix.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
public class StringDemo {
    public static void main(String[] args) {
        // === The pool / equality story ===
        String s1 = "hello";
        String s2 = "hello";                  // same pool entry as s1
        String s3 = new String("hello");      // explicit heap copy — different object

        System.out.println(s1 == s2);          // true  — same pool reference
        System.out.println(s1 == s3);          // false — different objects
        System.out.println(s1.equals(s3));     // true  — same characters

        // === Immutability ===
        String name = "Alice";
        name.toUpperCase();                    // RETURNS "ALICE" but DOESN'T modify name
        System.out.println(name);              // still "Alice" — String is immutable
        name = name.toUpperCase();             // now name points to a NEW "ALICE"

        // === The performance trap ===
        // BAD: O(n²) — every += creates a new String, copying all prior characters.
        String slow = "";
        for (int i = 0; i < 10_000; i++) slow = slow + i;

        // GOOD: O(n) — single buffer, append in place, one String created at the end.
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 10_000; i++) sb.append(i);
        String fast = sb.toString();
    }
}
```

What just happened: the equality demo proves the pool/heap split. The immutability lines show that `name.toUpperCase()` doesn't change `name` — it returns a new string, which we have to assign. The performance trap shows why `+=` in a loop is quadratic and why `StringBuilder` exists.

---

## 4. Build Up — Practical Patterns

### 4.1 Immutability — what and why

A `String` cannot be modified after construction. Every "modification" produces a new `String`.

Five reasons Java made `String` immutable:

| Reason | Explanation |
|--------|-------------|
| **Thread safety** | Shared freely between threads — no locks, no races. |
| **Security** | File paths, class names, URLs can't be tampered with after creation (no TOCTOU on a String). |
| **String pool** | Sharing only works because the shared entry can never change. |
| **`hashCode` caching** | Computed once, stored on the object, never recomputed. Crucial for HashMap key performance. |
| **Class design** | `String` is `final` — can't be subclassed to break invariants. |

```java
String name = "Alice";
name.toUpperCase();      // returns "ALICE" — `name` STILL points to "Alice"
name = name.toUpperCase();  // now `name` points to a new "ALICE"; old "Alice" eligible for GC
```

### 4.2 `==` vs `.equals` — reference vs content

```java
String a = "hello";
String b = "hello";
String c = new String("hello");

a == b;          // true  — same pool object (literals share)
a == c;          // false — c is a fresh heap object
a.equals(c);     // true  — same characters

// RULE: ALWAYS use .equals for String comparison. Never ==, no matter how clever you are.
```

`String.equals` checks `length`, then character-by-character. It also short-circuits on `==` first, so the cost is bounded.

### 4.3 The String pool — shared literals and `intern()`

```
       HEAP (since Java 7)
       ┌───────────────────────────────────┐
       │  POOL                              │
       │  ┌──────────────────────────────┐  │
       │  │ "hello"  "world"  "Java"     │  │  ← every string literal in your code
       │  └──────────────────────────────┘  │
       │                                    │
       │  Regular heap                      │
       │  ┌─────────┐ ┌─────────┐           │
       │  │ "hello" │ │ "hello" │           │  ← created by new String("hello")
       │  └─────────┘ └─────────┘           │
       └───────────────────────────────────┘
```

```java
String x = "hello";                 // pool
String y = new String("hello");     // heap
y == x;                             // false
y = y.intern();                     // y now points to the pool entry
y == x;                             // true
```

> 💡 **Java 7 moved the pool from PermGen to the regular heap.** Practical consequence: pooled strings can be garbage-collected when nothing references them. Pre-Java-7, leaking strings into PermGen via `intern()` could OutOfMemory the JVM.

### 4.4 StringBuilder vs StringBuffer — mutable buffers

| Type | Mutable | Thread-safe | Speed | When |
|------|:--:|:--:|:--:|---|
| `String` | ❌ | ✅ (because immutable) | slow for building | Single values, constants |
| `StringBuilder` | ✅ | ❌ | **fast** | Loops, building strings (default choice) |
| `StringBuffer` | ✅ | ✅ (`synchronized`) | slower | Legacy — practically obsolete |

```java
// Compiler optimisation — single-expression concat doesn't need manual SB
String full = first + " " + last;       // → StringBuilder, or invokedynamic on Java 9+

// In a LOOP, you must use StringBuilder explicitly
StringBuilder sb = new StringBuilder(estimateSize);    // pre-size if you can
for (var item : items) sb.append(item).append(",");
String csv = sb.toString();
```

### 4.5 Common String methods (the ones you need cold)

```java
String s = "  Hello, World!  ";
s.trim();                       // "Hello, World!"  — ASCII whitespace
s.strip();                      // same, Unicode-aware       (Java 11+)
s.stripLeading();               //                            (Java 11+)
s.stripTrailing();              //                            (Java 11+)
s.isBlank();                    // false                      (Java 11+)
s.substring(2, 7);              // "Hello"  — start inclusive, end EXCLUSIVE
s.indexOf("World");             // 9
s.contains("Hello");            // true
s.startsWith("  He");           // true
s.replace("World", "Java");     // replaces ALL occurrences
s.split(", ");                  // ["  Hello", "World!  "]
s.toCharArray();                // char[]
"ha".repeat(3);                 // "hahaha"                   (Java 11+)
"a\nb".lines();                 // Stream<String> ["a", "b"]  (Java 11+)
String.join(",", "a", "b", "c"); // "a,b,c"

// String.format / "...".formatted (Java 15+)
String.format("Total: %.2f", 12.5);    // "Total: 12.50"
"Total: %.2f".formatted(12.5);          // same
```

### 4.6 Text blocks (Java 15+)

```java
String json = """
        {
          "name": "Alice",
          "role": "trader"
        }
        """;
// Compiler strips the COMMON leading whitespace (8 spaces here) and preserves
// any indentation BEYOND that. Trailing newline is preserved unless you put a backslash:
String oneLine = """
        no \
        newline \
        between \
        these""";
```

### 4.7 `char[]` for sensitive data

A `String` may live in the pool indefinitely and isn't easily wiped. A `char[]` can be explicitly zeroed:

```java
char[] password = console.readPassword();
try {
    authenticate(password);
} finally {
    Arrays.fill(password, '\0');     // explicit wipe — gone from memory
}
```

This is why `Console.readPassword()` and JCA `KeySpec` APIs use `char[]`, not `String`. OWASP advises the same.

---

## 5. Going Deep — Interview-Level Material

### 5.1 Compact strings (Java 9+ `byte[]` + coder)

Pre-Java-9: `String` stored its data as `final char[]` — 2 bytes per character, even when every character was ASCII. For most programs (English-heavy text), that's 50% waste.

Java 9+ ([JEP 254](https://openjdk.org/jeps/254)): `String` now stores `final byte[]` plus a `byte coder` flag.

- If every character fits in **Latin-1** (1 byte) → coder = `LATIN1`, `byte[]` is exactly the bytes.
- Otherwise → coder = `UTF16`, `byte[]` is the same UTF-16 representation packed as 2-byte pairs.

Result: ASCII-heavy applications use roughly half the heap they used before, with no source change. Some operations (e.g. `charAt`) take a one-line conditional on the coder.

### 5.2 hashCode caching

`String.hashCode` is computed lazily on first call and cached in a private field. Because the value is immutable, the cache never invalidates. This is *the* reason `HashMap<String, ...>` lookups are so fast: a long key only pays the hash cost once.

```java
// Conceptually (real source has a small additional flag):
private int hash;
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        h = computeFromBytes();
        hash = h;
    }
    return h;
}
```

### 5.3 String concat compilation (Java 9+ `invokedynamic`)

What does `"x = " + x + ", y = " + y` compile to?

- **Java 8 and earlier:** desugared by the compiler to `new StringBuilder().append("x = ").append(x).append(", y = ").append(y).toString()`. Predictable, but inflexible — every concat allocates a new `StringBuilder` and a new array on each resize.
- **Java 9+** ([JEP 280](https://openjdk.org/jeps/280)): compiles to a single `invokedynamic` instruction calling `StringConcatFactory.makeConcat`. The JVM picks the most efficient strategy at runtime. Often allocates one final array of the right size in one shot, no resizes. Significantly faster.

> 💡 You almost never have to do anything: the compiler upgrades silently. The only catch — the strategy is selected per-callsite, so the first execution has a small linkage cost.

### 5.4 `intern()` mechanics and GC implications

`s.intern()` adds `s` to the pool if not present, then returns the pool's reference. Two consequences:

1. After `s = s.intern()`, `s == "hello"` becomes safe (assuming the literal exists). Useful for memory dedup of long-lived domain identifiers (currency codes, country codes).
2. **Pre-Java 7**: pool lived in PermGen. `intern()`'d strings could exhaust PermGen. **Post-Java 7**: pool is in the regular heap. Pooled strings are reclaimable when no references remain. Still, don't intern arbitrary user input — you risk holding onto strings far longer than the data flow requires.

### 5.5 Unicode, surrogates, and code points

A `char` in Java is a 16-bit UTF-16 code unit, **not** a code point. Characters outside the Basic Multilingual Plane (emoji, many Asian scripts, etc.) take **two** code units (a surrogate pair).

```java
String emoji = "🚀";
emoji.length();                    // 2 — two char (surrogate pair), not one
emoji.codePointCount(0, emoji.length());  // 1
emoji.codePoints().count();        // 1L  (Java 8+)
emoji.charAt(0);                   // high surrogate (0xD83D) — not the rocket
```

If you're processing user input or emoji, iterate over code points (`emoji.codePoints()`), not chars.

### 5.6 `substring` after Java 7 — no more shared array

Pre-Java-7: `substring(start, end)` returned a new `String` that **shared** the underlying `char[]` with the parent, just changing `offset` and `count`. Memory-efficient — but a tiny `substring` of a huge `String` kept the entire huge `char[]` alive. Classic leak.

Java 7 update: `substring` now copies the relevant range into a brand-new `byte[]`. Each substring is independent. The trade-off: cheap-substring micro-optimisations from old code books are wrong on modern JVMs.

### 5.7 The five rules for an immutable class

```java
public final class Period {                                     // 1. final class
    private final LocalDate start;                               // 2. private final fields
    private final LocalDate end;
    private final List<Holiday> holidays;

    public Period(LocalDate start, LocalDate end, List<Holiday> holidays) {  // 3. ctor only
        if (end.isBefore(start)) throw new IllegalArgumentException();
        this.start = start; this.end = end;
        this.holidays = List.copyOf(holidays);                   // 4. defensive copy IN
    }
    public LocalDate start() { return start; }                   // immutable types — safe to return
    public LocalDate end()   { return end; }
    public List<Holiday> holidays() { return holidays; }         // 5. List.copyOf result is unmodifiable
    // No setters, no mutators.                                  // 6. (corollary)
}
```

> 💡 Java 16+ records cover most of this for free, but you still need defensive copies for mutable components in a compact constructor. See [03 §5.3](./03_oop_language.md#53-records-java-16).

---

## 6. Memory Aids

### Decision tree: "should I use String, StringBuilder, or `+`?"

```
Building a string in a LOOP?
├── Yes → StringBuilder (pre-size if you can).
└── No
    ├── Single-expression concat (e.g. "x = " + x + ", y = " + y)?
    │   └── Use `+` directly. Compiler does the right thing.
    ├── Need thread-safe mutable buffer (rare)?
    │   └── StringBuffer (almost always you don't — single-threaded contexts dominate).
    └── It's just a constant or a one-off value?
        └── String literal.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Why is `String` immutable?" | Five reasons | "Thread safety, security, pool sharing, hashCode caching, final class." |
| "`==` vs `.equals` for strings?" | Reference vs content | "Always `.equals`. `==` only happens to work for pooled literals." |
| "What's `intern()`?" | Force into pool | "Returns the pool's reference. `s == "literal"` becomes safe afterwards." |
| "Why is `new String("x")` bad?" | Bypasses pool | "Forces a heap copy. Use the literal." |
| "How do I build an immutable class?" | Five rules | "final class, private final fields, ctor only, defensive copies in/out." |
| "Why `char[]` for passwords?" | Wipe-able | "Strings live in the pool indefinitely; char[] can be Arrays.fill('\0')." |
| "Compact strings?" | Java 9+ byte[]+coder | "Latin-1 strings stored as 1 byte/char; UTF-16 fallback for non-Latin." |

### Three anchor problems

1. **Strings are signs you can't edit.** Every "modification" creates a new sign.
2. **The pool is a shared drawer.** Literals share. `new String` doesn't.
3. **`==` is for "same object", `.equals` is for "same characters".** Almost always you want the second.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: Why is `String` immutable?
**A:** Five reasons: thread safety, security, the String pool only works if entries can't change, `hashCode` is cacheable for life, and `String` is `final` so subclasses can't break the invariant.

### Q2: What is the String pool?
**A:** A JVM-managed table of unique `String` literals, in the regular heap since Java 7 (was in PermGen before). Two `"hello"` literals refer to the same pool entry. `new String("hello")` makes a separate heap object. `intern()` forces a string into the pool.

### Q3: `==` vs `.equals` for Strings?
**A:** `==` compares references. `.equals` compares characters. `"hello" == "hello"` is true (shared pool entry). `new String("hello") == "hello"` is false. Use `.equals`.

### Q4: `String` vs `StringBuilder` vs `StringBuffer`?
**A:** `String` is immutable — concat creates new objects. `StringBuilder` is a mutable buffer, not thread-safe, fast — default choice for building. `StringBuffer` is the synchronised version of StringBuilder — practically obsolete.

### Q5: Why not concat strings in a loop?
**A:** Each `+=` allocates a fresh `String` and copies all prior characters → O(n²) total work. `StringBuilder.append` is O(n).

### Q6: What does `String.intern()` do?
**A:** If the pool already contains an equal string, return that reference. Otherwise add this string to the pool, return that reference. Lets you safely use `==` afterwards. Rarely needed in app code.

### Q7: How do you build an immutable class?
**A:** `final` class, `private final` fields, no setters, all values via constructor, defensive copies of mutable inputs and outputs. Records cover most of it (Java 16+) but you still need defensive copies for mutable components.

### Q8: Why use `char[]` for passwords?
**A:** A `String` may live in the pool indefinitely and isn't easily wiped. A `char[]` can be explicitly zeroed via `Arrays.fill(password, '\0')` after use. OWASP recommends `char[]` for credentials.

### Q9: What changed about strings in Java 9?
**A:** Compact strings ([JEP 254](https://openjdk.org/jeps/254)) — `String` now stores `byte[]` + `coder` flag. ASCII-heavy text takes ~half the memory. Plus `+` concat now compiles to `invokedynamic` ([JEP 280](https://openjdk.org/jeps/280)) — JVM picks the optimal strategy at runtime.

### Q10: What's a text block?
**A:** Java 15+ multi-line string literal: `"""` opens, `"""` closes. The compiler strips common leading whitespace. Useful for JSON, SQL, HTML literals.

### Q11: `String.length()` and emoji?
**A:** `length()` returns UTF-16 *code units*, not characters. Emoji and many Asian characters take a **surrogate pair** — two code units. Use `codePointCount(...)` or `codePoints()` to count actual characters.

### Q12: What's `substring`'s memory behaviour?
**A:** Pre-Java 7 it shared the parent's `char[]` — small substrings could pin entire huge strings in memory. Java 7+ copies into a fresh array — independent, no leak risk. The "substring is cheap" advice from old books is wrong on modern JVMs.

### Q13: How is `+` concat compiled?
**A:** Java 8 and earlier: desugared to a `StringBuilder` chain. Java 9+: a single `invokedynamic` to `StringConcatFactory.makeConcat`, JVM picks the strategy at runtime.

### Q14: `String.format` vs `+`?
**A:** `String.format` is slower (parses the format string, locale-aware) but clearer for templated output. `+` (and Java 15+'s `"…".formatted(...)`) is fine for simple cases.

### Q15: `equals` vs `equalsIgnoreCase`?
**A:** `equals` is exact byte-by-byte. `equalsIgnoreCase` performs locale-independent ASCII case folding (and Unicode-aware folding via `String.CASE_INSENSITIVE_ORDER`). For natural-language comparison, use `Collator`.

### Q16: Is `String` thread-safe?
**A:** Yes — it's immutable, so safe to share between any number of threads without synchronisation. That's a key reason for its design.

---

### Key Code Patterns

**Compare safely**
```java
"hello".equals(maybeNull);    // null-safe — use literal/known-non-null on the LEFT
Objects.equals(a, b);         // null-safe both sides
```

**Loop concatenation**
```java
StringBuilder sb = new StringBuilder(estimatedSize);
for (var x : items) sb.append(x).append(',');
String result = sb.toString();
```

**Defensive password handling**
```java
char[] pw = console.readPassword();
try { authenticate(pw); }
finally { Arrays.fill(pw, '\0'); }
```

**Text block for embedded JSON / SQL**
```java
String sql = """
        SELECT id, name
          FROM users
         WHERE active = true
        """;
```

---

## 8. Self-Test

**Easy**
- [ ] What does immutable mean for a `String`?
- [ ] Why use `.equals` instead of `==`?
- [ ] What does `name.toUpperCase()` do to `name`?
- [ ] When use `StringBuilder` over `+`?

**Medium**
- [ ] Trace through `s1 == s2`, `s1 == s3`, `s1.equals(s3)` in the §2 walkthrough.
- [ ] Why is `+=` in a loop O(n²)? What's the O(n) fix?
- [ ] What does `new String("hello")` do that the literal `"hello"` doesn't?
- [ ] List the five reasons `String` is immutable.
- [ ] Why use `char[]` instead of `String` for a password?

**Hard**
- [ ] Explain compact strings (Java 9+) — what's stored and why.
- [ ] What's the difference between `length()` and `codePointCount()`? When does it matter?
- [ ] What happened to `substring`'s memory behaviour in Java 7?
- [ ] How is `"a" + b + "c"` compiled in Java 9+ vs Java 8?
- [ ] Build an immutable `Period(start, end, holidays)` class using the five rules.

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Immutable** | Cannot be modified after construction. Every "modification" returns a new object. |
| **String literal** | A `"..."` in source code. The compiler stores it in the pool. |
| **String pool** | The shared drawer of unique String literals, in the regular heap since Java 7. |
| **`intern()`** | Add this string to the pool (or return the existing pool reference). |
| **`StringBuilder`** | Mutable, single-threaded buffer for building strings. |
| **`StringBuffer`** | Synchronised StringBuilder — practically obsolete. |
| **Code unit** | A 16-bit UTF-16 unit. Java's `char`. |
| **Code point** | An actual Unicode character. May take 1 or 2 code units. |
| **Surrogate pair** | Two `char`s representing one code point above 0xFFFF (e.g. emoji). |
| **Compact string** | Java 9+: storage as `byte[]` + `coder` flag. Latin-1 = 1 byte/char. |
| **Text block** | Java 15+ multi-line literal between triple-double-quote markers. |
| **Defensive copy** | Copy a mutable input on construction so callers can't mutate your state. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/05_strings_doubts.md) · [← Prev: 04 Exception Handling](./04_exceptions.md) · [Next: 06 Java 8+ →](./06_java8_plus.md)

[↑ Back to top](#05--strings--immutability)
