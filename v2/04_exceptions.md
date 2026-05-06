# 04 — Exception Handling

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/04_exceptions_doubts.md) · [← Prev: 03 OOP & Language](./03_oop_language.md) · [Next: 05 Strings →](./05_strings.md)
>
> **Priority:** 🟡 High · **Related topics:** [03 OOP](./03_oop_language.md) · [10 Testing](./10_testing.md) · [12 Spring Boot (error handling)](./12_spring_boot.md) · [16 Debugging](./16_debugging.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — what happens when something goes wrong
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — stack unwind, narrated
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — try / catch / finally and a custom exception
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 The hierarchy](#41-the-exception-hierarchy)
   - [4.2 Checked vs unchecked](#42-checked-vs-unchecked--when-to-use-each)
   - [4.3 try-with-resources](#43-try-with-resources-and-autocloseable-java-7)
   - [4.4 Exception chaining](#44-exception-chaining--preserve-the-cause)
   - [4.5 Multi-catch & precise rethrow](#45-multi-catch-and-precise-rethrow-java-7)
   - [4.6 Custom exceptions](#46-designing-custom-exceptions)
   - [4.7 Anti-patterns](#47-anti-patterns)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 The checked-exception debate](#51-the-checked-exception-debate-and-why-modern-frameworks-lean-unchecked)
   - [5.2 Suppressed exceptions](#52-suppressed-exceptions-and-getsuppressed)
   - [5.3 try-with-resources desugaring](#53-try-with-resources-desugaring)
   - [5.4 finally quirks](#54-finally-quirks-return-and-throw-from-finally)
   - [5.5 Cost of exceptions](#55-the-cost-of-throwing-stack-traces-and-the-jit)
   - [5.6 Helpful NPEs](#56-helpful-nullpointerexceptions-java-14)
   - [5.7 Global handlers (Spring)](#57-global-error-handling-the-spring-pattern)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Imagine a courier delivering a package to a 20-storey apartment building. He goes to flat 20A. *No one home.* Now what?

He doesn't drop the package and walk away. He climbs back down to 19A. *Also empty.* He tries 18A. *Tenant home, but won't sign.* Down to 17A. *Tenant signs and takes the package.* Done.

Two outcomes are possible:

- **Someone signs** for the package along the way → the package is delivered, the courier carries on with his round.
- **Nobody signs**, all the way down to the lobby → the courier leaves a "missed delivery" slip with the depot. Depot logs it. The depot supervisor decides what to do (re-attempt tomorrow? bounce back to sender?).

That's how exceptions work in Java. When code at one level can't handle a problem, it doesn't crash — it **throws** the problem to its caller. The caller can **catch** it and handle it, or pass it along by not catching it. The chain continues until either someone catches it, or the JVM itself catches it at the top of the thread (the "depot supervisor"), prints the stack trace, and ends the thread.

The smoke alarm metaphor reinforces: an exception is a signal, not a fatality. You decide whether to silence it (catch and recover), pass it along the wires (let it propagate), or let it ring until the building empties (let the JVM terminate the thread).

> Once you see the call stack as the apartment building and the exception as the package looking for a signer, every Java exception construct — `try`, `catch`, `throw`, `throws`, `finally`, try-with-resources — is just a way of telling Java who's home and who isn't.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

`main()` calls `processOrder()`, which calls `chargeCard()`, which throws because the card is declined.

```
main()
  └─ processOrder()
       └─ chargeCard()  ← throws CardDeclinedException
```

What happens, step by step:

| Step | What | Stack |
|:--:|---|---|
| 1 | `chargeCard` builds a `new CardDeclinedException(...)`. The constructor walks the current stack, captures the **stack trace**. | `[main → processOrder → chargeCard]` |
| 2 | `chargeCard` executes `throw exception`. The JVM begins unwinding. | unwinding from `chargeCard` |
| 3 | `chargeCard`'s frame pops. The JVM looks at `processOrder`'s `catch` blocks (if any) for a matching type. | `[main → processOrder]` |
| 4a | If `processOrder` has `catch (CardDeclinedException e)` → control jumps there, `e` is bound. **Building inhabited — package signed.** Normal execution resumes after the catch. | `[main → processOrder]` |
| 4b | If not — the exception keeps unwinding. `processOrder`'s frame pops. JVM looks at `main`'s catch blocks. Same logic. | `[main]` |
| 5 | If `main` doesn't catch either → `main`'s frame pops, JVM's default uncaught-exception handler prints the trace and ends the thread. **Nobody home — depot logs it.** | `[]` |

Two important wrinkles you'll see in the code:

- A `finally` block runs **on the way out**, whether the throw happened or not. Imagine taping a "leaving now" note inside every flat — even if you're rushing out because of an emergency, the note still gets stuck up. That's `finally`.
- A `try`-with-resources block has an **invisible** `finally` that calls `close()` on each resource — same flat-leaving note, written by the compiler.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
import java.io.*;

public class ExceptionDemo {
    // Custom unchecked exception with domain context.
    static class CardDeclinedException extends RuntimeException {
        final String cardLast4;
        CardDeclinedException(String cardLast4, String reason) {
            super("Card ending " + cardLast4 + " declined: " + reason);  // descriptive message
            this.cardLast4 = cardLast4;
        }
    }

    static void chargeCard(String last4) {
        // Pretend the gateway said "declined" — we throw with full context.
        throw new CardDeclinedException(last4, "INSUFFICIENT_FUNDS");
    }

    static void processOrder(String last4) {
        chargeCard(last4);                         // we don't catch — let it propagate up
    }

    public static void main(String[] args) {
        try (Reader r = new FileReader("orders.txt")) {  // try-with-resources: r.close() auto
            processOrder("1234");
        } catch (CardDeclinedException e) {              // domain-specific recovery
            System.err.println("Order failed: " + e.getMessage());
        } catch (IOException e) {                         // checked exception from FileReader
            System.err.println("Could not read orders: " + e.getMessage());
        } finally {
            System.out.println("processed one order attempt");  // ALWAYS runs
        }
    }
}
```

What just happened: the `Reader` is opened in the `try(...)` head; if anything inside the body throws, the compiler-generated cleanup closes the reader before propagating. The two `catch` blocks distinguish business logic failures (declined card) from infrastructure failures (file unreadable). `finally` runs in either case — useful for metric counters, audit logging, and similar "leaving the room" work.

---

## 4. Build Up — Practical Patterns

### 4.1 The exception hierarchy

```
                    Throwable
                   ╱          ╲
              Error            Exception
           (don't catch)      ╱          ╲
              │            (checked)   RuntimeException
              ├─ OutOfMemoryError    ├─ IOException     (unchecked)
              ├─ StackOverflowError  ├─ SQLException    ├─ NullPointerException
              └─ VirtualMachineError ├─ ParseException  ├─ IllegalArgumentException
                                     └─ Interrupted…    ├─ IllegalStateException
                                                         ├─ IndexOutOfBoundsException
                                                         ├─ ClassCastException
                                                         └─ ConcurrentModificationException
```

Three layers:
- **`Error`** — JVM-level catastrophes. Don't try to recover. (At most: log and exit.)
- **Checked `Exception`** — recoverable conditions the language *forces* you to acknowledge.
- **`RuntimeException`** (unchecked) — programming errors. Compiler does not enforce handling.

### 4.2 Checked vs unchecked — when to use each

| | Checked | Unchecked |
|---|---|---|
| Compiler enforces handling? | ✅ catch or `throws` | ❌ |
| Use for | "An external thing might fail; the caller should handle it" (file I/O, network, parse) | "A bug in the code's contract" (null arg, illegal state) |
| Examples | `IOException`, `SQLException`, `ParseException` | `IllegalArgumentException`, `NullPointerException`, `IllegalStateException` |
| Composes with lambdas/streams? | ❌ — leads to wrapping ceremony | ✅ |

> 💡 **Modern bias:** lean **unchecked**. Checked exceptions cause boilerplate (`try/catch` blocks that just rethrow), don't compose with `Stream`/`CompletableFuture` lambdas, and create coupling — adding a new `throws` to a low-level method ripples up through every caller. Spring, Hibernate, and most modern libraries throw unchecked exceptions on purpose.

### 4.3 try-with-resources and `AutoCloseable` (Java 7+)

Any object that implements `AutoCloseable` can go in `try(...)`. The compiler generates a hidden `finally` that calls `close()`, in **reverse declaration order**.

```java
// Verbose pre-Java-7 idiom — easy to get wrong
Connection conn = null;
PreparedStatement stmt = null;
ResultSet rs = null;
try {
    conn = ds.getConnection();
    stmt = conn.prepareStatement(sql);
    rs   = stmt.executeQuery();
    // ...
} finally {
    if (rs != null)   try { rs.close();   } catch (SQLException ignored) {}
    if (stmt != null) try { stmt.close(); } catch (SQLException ignored) {}
    if (conn != null) try { conn.close(); } catch (SQLException ignored) {}
}

// Modern equivalent — compiler generates everything correctly
try (Connection conn = ds.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql);
     ResultSet rs = stmt.executeQuery()) {
    // use rs
}
```

**Java 9+** lets you put an *effectively-final* variable directly in `try(...)`:

```java
Reader r = openReader();             // declared elsewhere, never reassigned → effectively final
try (r) { /* use r */ }              // legal in Java 9+
```

### 4.4 Exception chaining — preserve the cause

When you catch a low-level exception and throw a higher-level one, **always pass the original as the cause**. Otherwise the original stack trace is lost.

```java
// WRONG — root cause discarded
try { db.execute(query); }
catch (SQLException e) { throw new ServiceException("query failed"); }   // ❌

// RIGHT — chained
try { db.execute(query); }
catch (SQLException e) { throw new ServiceException("query failed", e); }
// In the log: ServiceException → caused by: SQLException → caused by: SocketException ...
```

`Throwable.getCause()` walks the chain.

### 4.5 Multi-catch and precise rethrow (Java 7+)

```java
// MULTI-CATCH — same handling for several types
try { parseTrade(input); }
catch (IOException | ParseException e) {              // e is implicitly final
    log.error("parse failed: {}", e.getMessage());
    throw new TradeProcessingException("parse failure", e);
}
// ⚠ Cannot mix a parent and its subclass in the same multi-catch (compile error).

// PRECISE RETHROW — compiler tracks the actual exception type
public void process() throws IOException, ParseException {     // narrow throws clause
    try { riskyOperation(); }
    catch (Exception e) {           // catch broadly for the log line
        log.error("failed", e);
        throw e;                    // compiler proves only IOException | ParseException can reach here
    }
}
```

### 4.6 Designing custom exceptions

Two reasons to write a custom exception:
1. The built-ins don't carry the **domain context** (account ID, trade ID, amounts).
2. Callers need a **specific catch block** for this kind of failure.

```java
public class InsufficientFundsException extends RuntimeException {
    private final String accountId;
    private final BigDecimal requested, available;

    public InsufficientFundsException(String accountId, BigDecimal requested, BigDecimal available) {
        super("Account %s: requested %s, available %s".formatted(accountId, requested, available));
        this.accountId = accountId;
        this.requested = requested;
        this.available = available;
    }
    public String getAccountId() { return accountId; }
    public BigDecimal getRequested() { return requested; }
    public BigDecimal getAvailable() { return available; }
}

// Throw early, with the full context the caller will need to handle this.
if (balance.compareTo(amount) < 0)
    throw new InsufficientFundsException(accountId, amount, balance);
```

> 💡 **Default to `extends RuntimeException`** unless the caller *must* be forced to handle it (and even then, think twice — see §5.1).

### 4.7 Anti-patterns

```java
// ❌ Swallowing
try { risky(); } catch (Exception e) { /* nothing */ }       // bug hides forever

// ❌ Catching too broadly
try { specific(); } catch (Throwable t) { /* … */ }          // also catches Errors

// ❌ Exceptions for flow control (slow + misleading)
try { return Integer.parseInt(input); }
catch (NumberFormatException e) { return defaultValue; }
// Better: a regex / explicit check.

// ❌ Returning from finally — overrides the try block's value
try { return computeValue(); } finally { return -1; }        // ALWAYS returns -1

// ❌ throws Exception on a public API — caller can't tell what to handle
public void doSomething() throws Exception { /* … */ }

// ❌ Logging AND rethrowing — log appears twice in the chain
catch (IOException e) { log.error("failed", e); throw e; }   // pick one or the other
```

---

## 5. Going Deep — Interview-Level Material

### 5.1 The checked-exception debate (and why modern frameworks lean unchecked)

Java is famously the only mainstream language with checked exceptions. The original case for them — "the compiler reminds you to handle expected failures" — runs into three real-world problems:

1. **Lambda incompatibility.** Functional interfaces like `Function<T, R>` and `Supplier<T>` don't declare any checked exceptions, so any code that throws a checked exception inside a `stream().map(...)` must wrap-rethrow as unchecked.
2. **Coupling propagation.** Adding `throws SQLException` to a low-level method forces every caller to either handle it or add `throws` themselves — the change ripples up and out.
3. **Boilerplate.** Vast quantities of `try { ... } catch (X e) { throw new Wrapper(e); }` exist purely to convert checked to unchecked, adding noise without value.

Spring, Hibernate, JDBI, and most modern frameworks deliberately use **unchecked** exceptions for their domain failures. The trade-off is conscious: you give up compiler-enforced handling in exchange for cleaner code and composability. Most of the time, the right place to handle an exception is at the *boundary* (HTTP response, message ack), not at the deepest call site — which is exactly where checked exceptions are loudest.

### 5.2 Suppressed exceptions and `getSuppressed`

Imagine: the body of a `try`-with-resources block throws, *and* the auto-generated `close()` also throws. Pre-Java 7, `finally`-based cleanup in this scenario would **replace** the original exception with the close-failure — losing the root cause.

Java 7's solution: the close-time exception is attached to the original via `Throwable.addSuppressed()`. The original is what the caller sees. The suppressed one is recoverable from `original.getSuppressed()`. This is one of the strongest reasons to prefer try-with-resources over `finally`-based cleanup.

```java
try (Reader r = open()) {
    riskyRead(r);                                    // throws IOException("read failed")
}
// In the catch handler one level up:
catch (IOException e) {
    e.getMessage();                                   // "read failed"
    Throwable[] sup = e.getSuppressed();              // [ IOException("close failed") ]
}
```

### 5.3 try-with-resources desugaring

The compiler rewrites:

```java
try (R r = expr) {  body  }
```

to roughly:

```java
R r = expr;
Throwable primary = null;
try {
    body
} catch (Throwable t) {
    primary = t;
    throw t;
} finally {
    if (r != null) {
        if (primary != null) {
            try { r.close(); } catch (Throwable t) { primary.addSuppressed(t); }
        } else {
            r.close();
        }
    }
}
```

That's where the suppressed-exception attachment comes from, and it's why you **can't** replicate try-with-resources cleanly in a hand-written `finally`.

### 5.4 `finally` quirks: return and throw from finally

Two booby traps:

```java
// 1. return in finally OVERRIDES try's return value.
int f() {
    try { return 1; }
    finally { return 2; }      // f() returns 2 — the 1 is silently discarded.
}

// 2. throw in finally REPLACES the original exception (and erases its stack trace).
int g() {
    try { throw new RuntimeException("primary"); }
    finally { throw new RuntimeException("from finally"); }
    // The caller sees "from finally". The "primary" exception is gone — not even suppressed.
}
```

Rule: **never `return` or `throw` from a `finally` block**. Use it only for cleanup (closing things you opened, restoring thread-locals, releasing locks, recording metrics).

### 5.5 The cost of throwing: stack traces and the JIT

Constructing an exception calls `fillInStackTrace()` natively, which is **the expensive part** — not the throw itself. For high-frequency control flow (e.g. millions of "not found" cases per second), the cost is real.

Two mitigations:
- For genuinely hot paths, return `Optional`, return a sentinel, or use a result type. Don't throw.
- For your own custom exception that doesn't need a stack trace (e.g. a flow-control exception inside a parser), override the constructor: `super(msg, cause, /*enableSuppression*/false, /*writableStackTrace*/false)`. The JVM skips `fillInStackTrace`. Use sparingly — debugging suffers.

### 5.6 Helpful NullPointerExceptions (Java 14+)

Pre-Java-14: `NullPointerException at com.foo.Bar.baz(Bar.java:42)`. Useful, but if line 42 is `a.b().c().d.e`, you don't know which dereference failed.

Java 14 (preview) and **Java 15+** (default-on): the JVM's NPE message tells you exactly which dereference: `Cannot invoke "Object.toString()" because the return value of "Bar.b()" is null`. Free, just by upgrading. Enabled by `-XX:+ShowCodeDetailsInExceptionMessages` (default on in Java 15+).

### 5.7 Global error handling: the Spring pattern

In a web application, you usually don't want each controller catching exceptions. Centralise the conversion from exception → HTTP response:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InsufficientFundsException.class)
    public ResponseEntity<ErrorResponse> handle(InsufficientFundsException e) {
        return ResponseEntity.status(422).body(new ErrorResponse("INSUFFICIENT_FUNDS", e.getMessage()));
    }

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(EntityNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }

    @ExceptionHandler(Exception.class)                       // catch-all — log + generic 500
    public ResponseEntity<ErrorResponse> handle(Exception e) {
        log.error("unexpected error", e);
        return ResponseEntity.status(500).body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}
```

More in [12 Spring Boot](./12_spring_boot.md).

---

## 6. Memory Aids

### Decision tree: "should I throw checked or unchecked?"

```
Is the failure a programming-error (null, illegal arg, illegal state)?
├── Yes → unchecked (RuntimeException subtype). Do NOT make caller handle a bug.
└── No
    ├── Is the failure expected and recoverable, AND callers should be reminded by the compiler?
    │   ├── Genuinely yes → checked (rare in modern Java).
    │   └── Probably no — most "expected failures" are better as unchecked + clear docs.
    └── Are you wrapping a checked exception from a lower layer (JDBC, IO)?
        └── Wrap as unchecked at the boundary. Don't propagate the noise.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Checked vs unchecked?" | Compiler-enforced vs not | "Modern bias: lean unchecked. Checked exceptions don't compose with lambdas." |
| "Why try-with-resources over finally?" | Suppressed exceptions | "Auto-close + the cause-preserving suppressed-exception mechanism." |
| "How do I keep the original cause?" | Exception chaining | "Pass it as the second constructor arg. `getCause()` walks the chain." |
| "What's wrong with `catch (Exception e)`?" | Too broad, hides bugs | "Catches NPE / ISE you didn't expect. Catch the most specific type." |
| "Should I `throw e` after logging?" | Log OR rethrow, not both | "Otherwise the same exception appears twice in the logs." |
| "Why not exceptions for flow control?" | `fillInStackTrace` is expensive | "Return an Optional or a result type." |

### Three anchor problems

1. **A throw walks UP the stack.** Frames pop until something catches.
2. **Always preserve the cause.** Otherwise root-cause analysis is impossible.
3. **`finally` is for cleanup, not for return values or new throws.**

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: Checked vs unchecked exceptions — hierarchy?
**A:** `Throwable` → `Error` (JVM catastrophes — don't catch) and `Exception`. `Exception` → checked (`IOException`, `SQLException`, `ParseException`) plus `RuntimeException` (unchecked: `NullPointerException`, `IllegalArgumentException`, `IllegalStateException`).

### Q2: When would you create a custom exception?
**A:** When built-ins don't convey domain meaning, or callers need a specific catch block. Include domain context (IDs, amounts) as fields. Default to `extends RuntimeException` unless the caller *must* handle it.

### Q3: How does try-with-resources work?
**A:** Resources in `try(...)` must implement `AutoCloseable`. Compiler generates a `finally` that calls `close()` in reverse declaration order. If both the body and `close()` throw, the close-time exception is suppressed (`addSuppressed`) onto the primary one, so the root cause is preserved.

### Q4: `finally` vs try-with-resources?
**A:** TWR is strictly better for resources — handles the suppressed-exception edge correctly, less verbose. Use `finally` only for non-resource cleanup (releasing locks, restoring ThreadLocal, recording metrics). Never `return` or `throw` from `finally`.

### Q5: What's wrong with `catch (Exception e)`?
**A:** Catches everything checked AND unchecked, including bugs you didn't anticipate (NPE, ISE). Hides root causes. Catch the most specific type instead. If you catch broadly to log, rethrow.

### Q6: `throw` vs `throws`?
**A:** `throw` — actually throws an exception instance (`throw new IllegalArgumentException("bad")`). `throws` — method signature declaring which checked exceptions might escape (`void read() throws IOException`).

### Q7: Multi-catch — what is it?
**A:** Catch multiple types in one block: `catch (IOException | ParseException e)`. The variable is implicitly final. Cannot list a type and its subtype together (compile error).

### Q8: Precise rethrow?
**A:** Java 7+: when you catch broadly and rethrow, the compiler tracks the actual types that could reach the throw. So `catch (Exception e) { throw e; }` inside a method whose body can only throw `IOException | ParseException` lets the method declare `throws IOException, ParseException` — not `throws Exception`.

### Q9: Exception chaining — what and why?
**A:** Wrapping a lower-level exception as the cause of a higher-level one: `throw new ServiceException("failed", ioe)`. Preserves the full stack trace. `Throwable.getCause()` walks the chain.

### Q10: Suppressed exceptions?
**A:** When a try-with-resources block's body throws AND `close()` also throws, the close-time exception is attached to the primary via `addSuppressed`. Recoverable via `getSuppressed()`. Without TWR, the close exception would silently replace the original.

### Q11: Can you catch `Error`?
**A:** Syntactically yes, semantically almost never. `OutOfMemoryError`, `StackOverflowError`, `VirtualMachineError` mean the JVM is in trouble. At most: catch, log, exit.

### Q12: Why are exceptions expensive?
**A:** `Throwable`'s constructor calls `fillInStackTrace()` (native) which walks the JVM stack and snapshots frames. For high-frequency throws on hot paths, the cost is real. Don't use exceptions for flow control.

### Q13: How do you turn off the stack trace for a custom exception?
**A:** Use the 4-arg constructor `super(message, cause, enableSuppression=false, writableStackTrace=false)`. JVM skips `fillInStackTrace`. Sparingly — debugging gets harder.

### Q14: What's a "helpful NPE"?
**A:** Java 14+ NullPointerException messages name exactly which dereference was null: `Cannot invoke "X.y()" because the return value of "A.b()" is null`. Default-on in Java 15+. Toggled by `-XX:+ShowCodeDetailsInExceptionMessages`.

### Q15: How do you handle exceptions globally in Spring?
**A:** `@RestControllerAdvice` class with `@ExceptionHandler` methods — central place to convert exception types into HTTP responses. Hierarchy fallback: a handler for the most specific type wins; an `Exception`-typed handler is the catch-all.

### Q16: Best-practice checklist?
**A:** (1) Catch specific, not broad. (2) Never swallow — log or rethrow. (3) TWR for all `AutoCloseable`. (4) Throw early (fail-fast at source), catch late (handle at boundary). (5) Include context in messages. (6) No exceptions for flow control. (7) Document `throws` in Javadoc.

---

### Key Code Patterns

**try-with-resources (multiple resources)**
```java
try (Connection c = ds.getConnection();
     PreparedStatement s = c.prepareStatement(sql);
     ResultSet rs = s.executeQuery()) {
    while (rs.next()) {/* … */}
}
```

**Exception chaining at a layer boundary**
```java
catch (SQLException e) { throw new RepositoryException("findById failed for id=" + id, e); }
```

**Custom domain exception with context**
```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String orderId) {
        super("Order not found: " + orderId);
    }
}
```

**Spring global handler**
```java
@RestControllerAdvice
class Errors {
    @ExceptionHandler(OrderNotFoundException.class)
    ResponseEntity<?> handle(OrderNotFoundException e) {
        return ResponseEntity.status(404).body(Map.of("error", "NOT_FOUND", "message", e.getMessage()));
    }
}
```

---

## 8. Self-Test

**Easy**
- [ ] What's the difference between `throw` and `throws`?
- [ ] Why use try-with-resources?
- [ ] What's the parent type of every exception?
- [ ] Should you catch `Error`?

**Medium**
- [ ] Walk through the courier analogy step by step. What's a `catch` block in the analogy? `finally`?
- [ ] When would you create a custom exception, and what should it carry?
- [ ] Why does try-with-resources beat hand-written `finally` cleanup?
- [ ] Why do modern frameworks lean toward unchecked exceptions?

**Hard**
- [ ] Explain suppressed exceptions and write the desugared form of a try-with-resources block.
- [ ] What's "precise rethrow"? Show a method signature where it changes the `throws` clause.
- [ ] Why is `return` from `finally` an anti-pattern? What about `throw`?
- [ ] How would you reduce the cost of throwing a custom exception used for flow control inside a parser?
- [ ] Write a `@RestControllerAdvice` with three handlers (specific business exception, not-found, catch-all).

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Throwable** | Anything you can `throw` — the root of the exception hierarchy. |
| **Error** | JVM-level failure (OOM, stack overflow). Don't catch. |
| **Checked exception** | The compiler forces you to catch or declare it. Subclass of `Exception` but not `RuntimeException`. |
| **Unchecked exception** | `RuntimeException` and its subclasses. Compiler doesn't enforce handling. |
| **Stack trace** | The list of method frames from `main` down to the throw site. |
| **Stack unwinding** | Popping frames one by one looking for a `catch`. |
| **Cause / chained exception** | A lower-level exception attached to a higher one via the constructor. |
| **Suppressed exception** | A close-time failure attached to a primary throw via `addSuppressed`. |
| **try-with-resources** | Java-7 syntax that auto-closes `AutoCloseable` resources. |
| **`AutoCloseable`** | Interface with `close()`. Required for try-with-resources. |
| **Multi-catch** | One `catch` block handling multiple exception types via `\|`. |
| **Precise rethrow** | Compiler tracks the actual subset of types that can reach a rethrow site. |
| **Helpful NPE** | Java 14+ NPE messages naming the exact null dereference. |
| **`@RestControllerAdvice`** | Spring class with `@ExceptionHandler` methods that convert exceptions to HTTP responses. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/04_exceptions_doubts.md) · [← Prev: 03 OOP & Language](./03_oop_language.md) · [Next: 05 Strings →](./05_strings.md)

[↑ Back to top](#04--exception-handling)
