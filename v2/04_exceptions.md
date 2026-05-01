# 04 — Exception Handling

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## 📚 Study Material

### 1. Exception Hierarchy

```
                    Throwable
                   ╱          ╲
              Error            Exception
           (don't catch)      ╱          ╲
           │                 │         RuntimeException
           ├─ OutOfMemoryError         (unchecked)
           ├─ StackOverflowError       │
           └─ VirtualMachineError      ├─ NullPointerException
                             │         ├─ IllegalArgumentException
                        (checked)      ├─ IllegalStateException
                             │         ├─ IndexOutOfBoundsException
                             ├─ IOException            ├─ ClassCastException
                             ├─ SQLException           ├─ UnsupportedOperationException
                             ├─ ParseException         └─ ConcurrentModificationException
                             └─ InterruptedException
```

**The split:**
- **Checked** (compile-time enforced): Must `catch` or declare `throws`. Represent *recoverable* conditions — file not found, network timeout, bad SQL.
- **Unchecked** (`RuntimeException` subtypes): Not enforced. Represent *programming errors* — null dereference, bad argument, invalid state.
- **Error**: JVM-level catastrophes. Don't catch (except for logging/cleanup in very specific cases).

💡 **Modern preference:** Use unchecked exceptions. Checked exceptions create coupling, don't compose with lambdas/streams, and lead to meaningless catch-and-rethrow boilerplate.

### 2. Try-With-Resources — How It Actually Works

```java
// BEFORE Java 7 — verbose, error-prone
Connection conn = null;
PreparedStatement stmt = null;
ResultSet rs = null;
try {
    conn = dataSource.getConnection();
    stmt = conn.prepareStatement("SELECT * FROM trades");
    rs = stmt.executeQuery();
    // process results
} catch (SQLException e) {
    throw new ServiceException("Query failed", e);
} finally {
    // Must close in reverse order, each with its own try/catch
    if (rs != null) try { rs.close(); } catch (SQLException ignored) {}
    if (stmt != null) try { stmt.close(); } catch (SQLException ignored) {}
    if (conn != null) try { conn.close(); } catch (SQLException ignored) {}
}

// AFTER Java 7 — compiler handles everything
try (Connection conn = dataSource.getConnection();           // AutoCloseable
     PreparedStatement stmt = conn.prepareStatement(sql);    // closed in REVERSE order
     ResultSet rs = stmt.executeQuery()) {                   // if close() throws →
    // process results                                       // attached as suppressed
} catch (SQLException e) {
    throw new ServiceException("Query failed", e);
    // e.getSuppressed() — any exceptions from close() calls
}
```

**Suppressed exception mechanism:**
```java
// If BOTH the try body AND close() throw:
// 1. The try-body exception is the PRIMARY exception (thrown to caller)
// 2. The close() exception is SUPPRESSED (attached via addSuppressed)
// 3. You can access it: primaryException.getSuppressed()
// This is superior to finally-based cleanup where close() would REPLACE the original exception
```

### 3. Custom Exceptions — When and How

```java
// Include context — makes debugging in production 10x faster
public class InsufficientFundsException extends RuntimeException {
    private final String accountId;
    private final BigDecimal requested;
    private final BigDecimal available;
    
    public InsufficientFundsException(String accountId, BigDecimal requested, BigDecimal available) {
        super(String.format("Account %s: requested %s but only %s available",
              accountId, requested, available));     // descriptive message
        this.accountId = accountId;
        this.requested = requested;
        this.available = available;
    }
    
    // Getters for programmatic access (e.g., error response body)
    public String getAccountId() { return accountId; }
    public BigDecimal getRequested() { return requested; }
    public BigDecimal getAvailable() { return available; }
}

// Usage — throw early with full context
public void withdraw(String accountId, BigDecimal amount) {
    BigDecimal balance = getBalance(accountId);
    if (balance.compareTo(amount) < 0) {
        throw new InsufficientFundsException(accountId, amount, balance);  // THROW EARLY
    }
    // proceed
}
```

**When to create custom exceptions:**
- Built-in exceptions don't convey domain meaning
- You need to attach domain-specific context (account ID, trade ID, amounts)
- Your API needs specific catch blocks for different error types
- **Extend `RuntimeException`** (unchecked) unless the caller *must* handle it

### 4. Exception Chaining — Preserving Root Cause

```java
// WRONG — loses the original stack trace
try {
    database.execute(query);
} catch (SQLException e) {
    throw new ServiceException("Query failed");   // ❌ original cause is LOST
}

// RIGHT — chain the original exception
try {
    database.execute(query);
} catch (SQLException e) {
    throw new ServiceException("Query failed", e);   // ✅ original cause preserved
    // In logs: ServiceException → caused by: SQLException → caused by: SocketException
    // getCause() walks the chain
}
```

### 5. Multi-Catch and Rethrowing

```java
// Multi-catch (Java 7+) — one block handles multiple types
try {
    parseTrade(input);
} catch (IOException | ParseException e) {    // e is implicitly final
    log.error("Failed to parse trade: {}", e.getMessage());
    throw new TradeProcessingException("Parse failure", e);
}
// ⚠️ Can't catch parent and child in same multi-catch (compiler error)
// catch (Exception | IOException e) — ❌ IOException is a subtype of Exception

// Precise rethrow (Java 7+) — compiler tracks actual exception type
public void process() throws IOException, ParseException {
    try {
        riskyOperation();
    } catch (Exception e) {      // catches broadly
        log.error("Failed", e);
        throw e;                 // compiler knows only IOException | ParseException can reach here
    }                            // so the throws clause is precise, not "throws Exception"
}
```

### 6. Best Practices & Anti-Patterns

```java
// ❌ ANTI-PATTERN: Swallowing exceptions
try { riskyOperation(); }
catch (Exception e) { /* empty — bug hides forever */ }

// ❌ ANTI-PATTERN: Catching too broadly
try { specificOperation(); }
catch (Exception e) { /* catches NullPointerException you didn't expect */ }

// ❌ ANTI-PATTERN: Using exceptions for flow control
try {
    int value = Integer.parseInt(input);  // expensive: fills stack trace
} catch (NumberFormatException e) {
    value = defaultValue;                 // use validation instead
}
// ✅ BETTER: if (input.matches("-?\\d+")) — check first, don't rely on exceptions

// ❌ ANTI-PATTERN: Returning from finally
try { return computeValue(); }
finally { return -1; }  // ❌ ALWAYS returns -1 — overrides try block's return

// ✅ BEST PRACTICES:
// 1. Catch specific, not broad
// 2. Never swallow — at minimum log.error("msg", e)
// 3. Throw early (fail fast at the source), catch late (handle at the boundary)
// 4. Include context in messages (IDs, values, state)
// 5. Use try-with-resources for ALL AutoCloseable resources
// 6. Don't use exceptions for flow control — they fill stack traces (expensive)
// 7. Document thrown exceptions in Javadoc
```

### 7. Spring Exception Handling Pattern

```java
// Centralised error handling — one place for all error formatting
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(InsufficientFundsException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientFunds(InsufficientFundsException e) {
        return ResponseEntity.status(422).body(
            new ErrorResponse("INSUFFICIENT_FUNDS", e.getMessage())
        );
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException e) {
        return ResponseEntity.status(404).body(
            new ErrorResponse("NOT_FOUND", e.getMessage())
        );
    }
    
    @ExceptionHandler(Exception.class)    // catch-all — last resort
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("Unexpected error", e);  // ALWAYS log unexpected exceptions
        return ResponseEntity.status(500).body(
            new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred")
            // Don't leak internal details to the client
        );
    }
}
```

---

## Rapid-Fire Q&A

### Q1: Checked vs Unchecked exceptions — hierarchy?
**A:** `Throwable` → `Error` (JVM-level: `OutOfMemoryError`, `StackOverflowError` — don't catch) and `Exception`. `Exception` → Checked (`IOException`, `SQLException`) + `RuntimeException` (unchecked: `NullPointerException`, `IllegalArgumentException`, `IllegalStateException`).

### Q2: When would you create a custom exception?
**A:** When built-in exceptions don't convey domain meaning. E.g., `InsufficientFundsException extends RuntimeException`. Include context: account ID, attempted amount, available balance. Extend `RuntimeException` (unchecked) unless caller must handle it.

### Q3: Try-with-resources — how does it work?
**A:** Resources declared in `try(...)` must implement `AutoCloseable`. Compiler generates `finally` with `close()` calls in reverse declaration order. If both try body and `close()` throw, the close exception is suppressed (attached via `addSuppressed`).

### Q4: `finally` vs `try-with-resources`?
**A:** `try-with-resources` is strictly better for resource cleanup — handles the suppressed-exception edge case correctly, is less verbose. Use `finally` only for non-resource cleanup (resetting state, logging). Never use `finally` to return a value — it overrides the try block's return.

### Q5: What happens if you catch `Exception` broadly?
**A:** Catches all checked AND unchecked exceptions — too broad. Hides bugs (catches `NullPointerException` you didn't expect). Catch the most specific exception possible. If you must catch broadly, re-throw or log with the full stack trace.

### Q6: `throw` vs `throws`?
**A:** `throw` — actually throw an exception instance (`throw new IllegalArgumentException("bad")`). `throws` — method signature declaration that checked exceptions may be thrown (`void read() throws IOException`).

### Q7: Can you catch multiple exceptions in one block?
**A:** Yes. Multi-catch: `catch (IOException | ParseException e)`. The variable `e` is implicitly `final`. Cannot catch parent and child in same multi-catch (compiler error).

### Q8: What's exception chaining?
**A:** Wrapping a lower-level exception as the cause of a higher-level one: `throw new ServiceException("Failed", ioe)`. Preserves the full stack trace. Accessed via `getCause()`.

### Q9: Best practices for exception handling?
**A:** (1) Catch specific, not broad. (2) Don't swallow exceptions — at minimum log them. (3) Use try-with-resources for all resources. (4) Throw early, catch late. (5) Include context in exception messages. (6) Don't use exceptions for flow control (expensive — fills stack trace).

---

## Can you answer these cold?

- [ ] Exception hierarchy — Throwable → Error / Exception → RuntimeException
- [ ] Checked vs unchecked — when each is appropriate
- [ ] Try-with-resources — AutoCloseable, suppressed exceptions
- [ ] Custom exception — when, how, what to include
- [ ] Multi-catch syntax

[← Back to Index](./00_INDEX.md)
