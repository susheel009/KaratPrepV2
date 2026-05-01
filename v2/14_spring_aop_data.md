# 14 — Spring AOP & Data

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

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
