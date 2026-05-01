# 14 — Spring AOP & Data

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

---

## 🟢 Start Here — AOP & Database Access in Plain English

### What is AOP (Aspect-Oriented Programming)?

Imagine every method in your app needs to: (1) log when it starts, (2) check permissions, (3) measure how long it takes. You could add that code to **every single method** — but that's messy and repetitive.

**AOP lets you define that code ONCE and apply it everywhere automatically.**

Think of it like a security camera at a store entrance. You don't install one inside every aisle — you put one at the door (the *aspect*), and it captures everyone who enters (*all methods*).

```java
// WITHOUT AOP — logging scattered everywhere
public void processOrder(Order o) {
    log.info("Starting processOrder");     // repeated in every method
    // ... business logic ...
    log.info("Finished processOrder");     // repeated in every method
}

// WITH AOP — logging defined ONCE
@Before("execution(* com.citi.service.*.*(..))")   // "before any method in service package"
public void logEntry(JoinPoint jp) {
    log.info("Starting {}", jp.getSignature().getName());
}
// Now EVERY service method gets automatic logging — zero changes to business code
```

### What is JPA / Spring Data?

**JPA (Java Persistence API)** is how Java talks to databases. Instead of writing SQL by hand, you define Java classes and JPA translates them to database tables.

**Spring Data JPA** makes it even easier — you just define an interface, and Spring writes the SQL for you.

```java
// Your Java class (entity) → maps to a database table
@Entity
public class Trade {
    @Id
    private Long id;
    private String symbol;     // maps to a column called "symbol"
    private int quantity;
}

// Your repository interface → Spring generates ALL the SQL
public interface TradeRepository extends JpaRepository<Trade, Long> {
    List<Trade> findBySymbol(String symbol);   // Spring generates: SELECT * FROM trade WHERE symbol = ?
}

// Usage — no SQL needed!
List<Trade> appleTrades = tradeRepo.findBySymbol("AAPL");
```

### What's the N+1 problem? (the most common performance bug)

If you load 100 orders and each order has items, JPA might run **101 queries** (1 for orders + 100 for each order's items). The fix is loading everything in one query with `JOIN FETCH`.

> Key takeaway: AOP = define cross-cutting logic once, apply everywhere. JPA = Java objects ↔ database tables. Spring Data = write an interface, get SQL for free.

---

## 📚 Study Material — AOP

### 1. What AOP Solves

Without AOP, cross-cutting concerns pollute every service method:
```java
// ❌ WITHOUT AOP — logging, timing, security scattered everywhere
public Trade executeTrade(TradeRequest req) {
    log.info("Entering executeTrade");              // logging
    if (!securityContext.hasRole("TRADER")) throw…; // security
    long start = System.nanoTime();                 // timing
    try {
        Trade result = doExecute(req);
        long elapsed = System.nanoTime() - start;
        metrics.record("trade.execute", elapsed);   // metrics
        log.info("Exiting executeTrade");
        return result;
    } catch (Exception e) {
        log.error("Failed", e);                     // error logging
        throw e;
    }
}

// ✅ WITH AOP — business logic is clean
@Timed @Secured("TRADER") @Audited
public Trade executeTrade(TradeRequest req) {
    return doExecute(req);                          // pure business logic
}
// Cross-cutting concerns are defined ONCE in aspects
```

### 2. AOP Terminology

| Term | Meaning | Example |
|------|---------|---------|
| **Aspect** | Module containing cross-cutting logic | `@Aspect class LoggingAspect` |
| **Join Point** | A point in execution (always a method call in Spring) | `executeTrade()` |
| **Pointcut** | Expression selecting join points | `execution(* com.citi.service.*.*(..))` |
| **Advice** | Code that runs at a join point | `@Before`, `@After`, `@Around` |
| **Weaving** | Applying aspects to targets | At runtime via proxies (Spring) |

### 3. Advice Types with Code

```java
@Aspect
@Component
public class AuditAspect {
    
    // BEFORE — runs before the method
    @Before("execution(* com.citi.service.TradeService.*(..))")
    public void logEntry(JoinPoint jp) {
        log.info("Calling {} with args {}", jp.getSignature().getName(), jp.getArgs());
    }
    
    // AFTER RETURNING — runs after successful return
    @AfterReturning(pointcut = "execution(* com.citi.service.*.*(..))", returning = "result")
    public void logResult(JoinPoint jp, Object result) {
        log.info("{} returned {}", jp.getSignature().getName(), result);
    }
    
    // AFTER THROWING — runs after exception
    @AfterThrowing(pointcut = "execution(* com.citi.service.*.*(..))", throwing = "ex")
    public void logError(JoinPoint jp, Exception ex) {
        log.error("{} threw {}", jp.getSignature().getName(), ex.getMessage());
    }
    
    // AFTER — runs always (like finally)
    @After("execution(* com.citi.service.*.*(..))")
    public void logExit(JoinPoint jp) {
        log.info("Exited {}", jp.getSignature().getName());
    }
    
    // AROUND — most powerful: wraps the method, controls proceed()
    @Around("@annotation(com.citi.Timed)")          // targets methods with @Timed annotation
    public Object timeMethod(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed();           // ← actually calls the target method
            return result;
        } finally {
            long elapsed = System.nanoTime() - start;
            metrics.record(pjp.getSignature().getName(), elapsed);
        }
    }
}
```

### 4. Pointcut Expressions

```java
// Method execution in specific package
execution(* com.citi.service.*.*(..))
//  ↑ return type (any)  ↑ any class  ↑ any method  ↑ any args

// Only public methods
execution(public * com.citi.service.*.*(..))

// Methods returning void
execution(void com.citi.service.*.*(..))

// Specific method name
execution(* com.citi.service.TradeService.execute*(..))

// By annotation on method
@annotation(com.citi.Timed)

// By annotation on class
@within(org.springframework.stereotype.Service)

// All classes in package (and sub-packages)
within(com.citi.service..*)

// Combine with && || !
execution(* com.citi.service.*.*(..)) && @annotation(Timed)
```

### 5. Spring AOP Proxy Mechanism

```
Caller → Proxy → Advice (Before/Around) → Target Method → Advice (After) → Caller

Two proxy types:
1. JDK Dynamic Proxy — if target implements an interface
2. CGLIB Proxy — if target is a concrete class (creates a SUBCLASS)
   Spring Boot defaults to CGLIB (proxyTargetClass=true)
```

⚠️ **Self-invocation trap (CRITICAL):**
```java
@Service
public class OrderService {
    @Timed
    public void processOrder(Order o) { /* timed */ }
    
    public void batchProcess(List<Order> orders) {
        for (Order o : orders) {
            this.processOrder(o);   // ❌ `this` is the REAL object, not the proxy
                                    // @Timed advice WILL NOT run
        }
    }
}
// The proxy only intercepts calls from OUTSIDE the bean
// Internal calls (this.method()) go directly to the real object
```

---

## 📚 Study Material — Spring Data / JPA

### 6. JPA Architecture

```
Your Code → Repository Interface → Spring Data JPA → JPA (Hibernate) → JDBC → Database

Spring Data JPA generates the repository implementation at startup.
You write ZERO boilerplate SQL for standard CRUD.
```

```java
// Define entity
@Entity
@Table(name = "trades")
public class Trade {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String symbol;
    
    private int quantity;
    
    @Enumerated(EnumType.STRING)        // store enum as string, not ordinal
    private Side side;
    
    @ManyToOne(fetch = FetchType.LAZY)   // default for @ManyToOne is EAGER — override to LAZY
    @JoinColumn(name = "trader_id")
    private Trader trader;
    
    @Version                             // optimistic locking field
    private Long version;
}

// Define repository — Spring generates implementation
public interface TradeRepository extends JpaRepository<Trade, Long> {
    List<Trade> findBySymbolAndSide(String symbol, Side side);  // derived query
    
    @Query("SELECT t FROM Trade t WHERE t.quantity > :min")     // JPQL
    List<Trade> findLargeOrders(@Param("min") int min);
    
    @Query(value = "SELECT * FROM trades WHERE symbol = ?1", nativeQuery = true)  // native SQL
    List<Trade> findBySymbolNative(String symbol);
    
    Page<Trade> findByTrader(Trader trader, Pageable pageable); // pagination
}
```

### 7. Entity Lifecycle (Persistence Context)

```
    new Trade()          repo.save(trade)        tx commits / detach
        │                      │                       │
   TRANSIENT ──────→ MANAGED (PERSISTENT) ──────→ DETACHED
    (not tracked)     (changes auto-flushed)     (no longer tracked)
                           │
                      repo.delete()
                           │
                      REMOVED
```

```java
@Transactional
public void updateTrade(Long id) {
    Trade trade = repo.findById(id).orElseThrow();  // trade is MANAGED
    trade.setQuantity(200);                          // DIRTY CHECKING: JPA auto-detects change
    // NO explicit save() needed!                    // Hibernate generates UPDATE at flush/commit
}
// After transaction ends → trade becomes DETACHED
// trade.setQuantity(300);  ← this change is LOST (not tracked anymore)
```

### 8. The N+1 Problem

```java
// PROBLEM:
@Entity
public class Trader {
    @OneToMany(mappedBy = "trader", fetch = FetchType.LAZY)
    private List<Trade> trades;
}

// This code generates N+1 queries:
List<Trader> traders = traderRepo.findAll();    // 1 query: SELECT * FROM traders
for (Trader t : traders) {
    t.getTrades().size();                       // N queries: SELECT * FROM trades WHERE trader_id = ?
}
// 100 traders = 101 queries!

// FIX 1: JOIN FETCH (JPQL)
@Query("SELECT t FROM Trader t JOIN FETCH t.trades")
List<Trader> findAllWithTrades();
// 1 query with JOIN — loads everything at once

// FIX 2: @EntityGraph
@EntityGraph(attributePaths = {"trades"})
List<Trader> findAll();
// Tells JPA to eagerly fetch 'trades' for this specific query

// FIX 3: Batch fetching
@BatchSize(size = 25)  // on the collection
// Loads trades in batches: WHERE trader_id IN (?, ?, ..., ?)
// 100 traders = 1 + 4 queries (batches of 25)
```

### 9. Optimistic vs Pessimistic Locking

```java
// OPTIMISTIC — version-based (no database locks, best for low contention)
@Entity
public class Account {
    @Version                                    // JPA manages this automatically
    private Long version;
    private BigDecimal balance;
}
// UPDATE accounts SET balance=?, version=version+1 WHERE id=? AND version=?
// If version doesn't match → OptimisticLockException (someone else updated first)
// Handle: retry the operation, or inform the user

// PESSIMISTIC — database-level lock (use for high contention)
@Lock(LockModeType.PESSIMISTIC_WRITE)          // SELECT ... FOR UPDATE
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdForUpdate(@Param("id") Long id);
// Blocks other transactions from reading/writing until lock is released
// Use for: bank transfers, inventory reservation — can't afford lost updates
```

### 10. Transaction Propagation

```java
// REQUIRED (default) — join existing tx or create new
@Transactional
public void placeOrder(Order order) {
    validateOrder(order);        // runs in SAME transaction
    saveOrder(order);            // runs in SAME transaction
    sendNotification(order);     // what if this needs its OWN transaction?
}

// REQUIRES_NEW — always create new (suspend current)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void audit(String action) {
    // Runs in a NEW transaction — committed even if caller's tx rolls back
    // Use for: audit logs, notifications that MUST persist regardless
}

// Example: transfer with independent audit
@Transactional
public void transfer(String from, String to, BigDecimal amount) {
    debit(from, amount);         // same tx
    credit(to, amount);          // same tx
    audit("TRANSFER", from, to); // REQUIRES_NEW → saved even if transfer fails after this
}
```

### 11. Spring Caching

```java
@EnableCaching                                   // on @Configuration class

@Service
public class TradeService {
    
    @Cacheable("trades")                         // cache result by method args
    public Trade findById(Long id) {
        return repo.findById(id).orElseThrow();  // DB hit only on cache miss
    }
    
    @CachePut("trades")                          // always execute, update cache
    public Trade updateTrade(Long id, Trade trade) {
        return repo.save(trade);                 // updates DB AND cache
    }
    
    @CacheEvict("trades")                        // remove from cache
    public void deleteTrade(Long id) {
        repo.deleteById(id);
    }
    
    @CacheEvict(value = "trades", allEntries = true)  // clear entire cache
    @Scheduled(fixedRate = 3600000)
    public void evictAll() {}
}

// Cache backends:
// Default: ConcurrentHashMap (in-memory, single JVM)
// Production: Caffeine (high-perf local), Redis (distributed), EhCache
```

---

## Rapid-Fire Q&A — AOP

### Q1: What's AOP and what problem does it solve?
**A:** Aspect-Oriented Programming separates cross-cutting concerns (logging, security, transactions, caching) from business logic. Without AOP, you'd scatter `log.info()` and `try/catch` through every service method. With AOP, you define it once in an aspect.

### Q2: Key AOP terms?
**A:** **Aspect**: the cross-cutting concern module. **JoinPoint**: a point in execution (method call, field access). **Pointcut**: expression selecting which JoinPoints to target. **Advice**: code that runs at a JoinPoint (before, after, around). **Weaving**: applying aspects to target objects.

### Q3: Advice types?
**A:** `@Before` — runs before method. `@After` — runs after (regardless of outcome). `@AfterReturning` — after successful return. `@AfterThrowing` — after exception. `@Around` — wraps method, you control `proceed()`. Around is most powerful but most dangerous.

### Q4: How does Spring AOP work internally?
**A:** Proxy-based. For interfaces: JDK dynamic proxy. For classes: CGLIB subclass proxy. The proxy intercepts calls and applies advice. **Limitation:** self-invocation (`this.method()`) bypasses the proxy — advice won't run. Same issue as `@Transactional`.

### Q5: Pointcut expression examples?
**A:** `execution(* com.citi.service.*.*(..))` — any method in service package. `@annotation(com.citi.Timed)` — methods annotated with `@Timed`. `within(com.citi.service..*)` — any class in service and sub-packages.

---

## Rapid-Fire Q&A — Spring Data / JPA

### Q6: What's JPA? What's Hibernate?
**A:** JPA = Java Persistence API — the specification (interfaces). Hibernate = the most common JPA implementation. Spring Data JPA = repository abstraction on top of JPA. You define `interface UserRepository extends JpaRepository<User, Long>` — Spring generates the implementation.

### Q7: What's the N+1 problem?
**A:** Lazy-loaded relationships execute 1 query for the parent + N queries for each child. E.g., load 100 orders, then 100 queries for items. Fix: `JOIN FETCH` in JPQL, `@EntityGraph`, or `FetchType.EAGER` (but eager has its own problems).

### Q8: `@Entity` lifecycle — managed, detached, transient?
**A:** **Transient**: new object, not tracked. **Managed**: attached to persistence context, changes auto-flushed. **Detached**: was managed, but persistence context closed (e.g., after transaction ends). **Removed**: marked for deletion.

### Q9: Optimistic vs pessimistic locking in JPA?
**A:** **Optimistic**: `@Version` field. On update, JPA checks version — if changed since read, throws `OptimisticLockException`. No database locks. Good for low-contention. **Pessimistic**: `@Lock(LockModeType.PESSIMISTIC_WRITE)` — database-level `SELECT FOR UPDATE`. Blocks other transactions. Use for high-contention.

### Q10: Spring Data query methods?
**A:** Method name → query: `findByEmailAndActive(String email, boolean active)`. Custom: `@Query("SELECT u FROM User u WHERE u.dept = :dept")`. Native: `@Query(value = "SELECT * FROM users WHERE...", nativeQuery = true)`. Pagination: `Page<User> findByDept(String dept, Pageable pageable)`.

### Q11: Transaction propagation?
**A:** `REQUIRED` (default): join existing tx or create new. `REQUIRES_NEW`: always new tx (suspends current). `SUPPORTS`: use existing if present, none otherwise. `MANDATORY`: must have existing tx. `NOT_SUPPORTED`: suspends current tx. `NEVER`: throws if tx exists.

### Q12: What's Spring caching?
**A:** `@Cacheable("users")` — caches return value by args. `@CacheEvict("users")` — clears cache. `@CachePut` — always runs method, updates cache. Backed by: ConcurrentHashMap (default), Caffeine, Redis, EhCache. `@EnableCaching` required.

---

## Can you answer these cold?

- [ ] AOP advice types — Before, After, AfterReturning, AfterThrowing, Around
- [ ] Spring AOP proxy mechanism — why self-invocation doesn't work
- [ ] N+1 problem — what it is, three fixes
- [ ] Optimistic vs pessimistic locking — when each
- [ ] Transaction propagation — REQUIRED vs REQUIRES_NEW
- [ ] Spring caching — `@Cacheable`, `@CacheEvict`

[← Back to Index](./00_INDEX.md)
