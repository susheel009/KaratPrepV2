# 12 — Spring Boot

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## 🟢 Start Here — Spring Boot in Plain English

### What is Spring Boot?

**Spring Boot = Spring made easy.** Spring (the framework) is powerful but requires a lot of configuration. Spring Boot gives you **sensible defaults** so you can start building immediately.

Think of it like buying furniture:
- **Spring** = you buy raw wood, screws, and tools. You build everything from scratch.
- **Spring Boot** = you buy from IKEA. Pre-cut, instructions included, ready in 30 minutes.

### What does Spring Boot give you?

```
✅ Embedded web server (Tomcat) — no separate server needed
✅ Auto-configuration — "you added a database driver? I'll configure a connection pool for you"
✅ Starter dependencies — one dependency gives you everything you need
✅ application.yml — put all your settings in one file
✅ Actuator — health checks and monitoring built in
```

### The simplest Spring Boot app

```java
@SpringBootApplication         // this one annotation does everything
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);   // starts the app!
    }
}

@RestController                // handles HTTP requests
class HelloController {
    @GetMapping("/hello")      // when someone visits /hello
    String hello() {
        return "Hello, World!";  // return this text
    }
}
// That's it! Run the app, visit http://localhost:8080/hello → "Hello, World!"
```

### What's @Transactional?

When working with a database, you want operations to be **all-or-nothing**. If you're transferring money from A to B, either both happen or neither does.

```java
@Transactional  // Spring says: "wrap this in a database transaction"
public void transfer(Account from, Account to, double amount) {
    from.debit(amount);     // step 1: take money out
    to.credit(amount);      // step 2: put money in
    // If step 2 fails → step 1 is automatically UNDONE (rolled back)
}
```

> Key takeaway: Spring Boot = Spring with batteries included. Auto-configuration handles the boring stuff. You just write your business logic.

---

## 📚 Study Material

### 1. Spring Boot = Spring + Opinionated Defaults

```
Spring Framework (core)  →  YOU configure everything (XML, JavaConfig, web.xml)
Spring Boot              →  Auto-configures based on classpath + conventions

What Spring Boot gives you:
✅ Embedded server (Tomcat/Jetty/Undertow) — no WAR deployment
✅ Starter dependencies — curated, version-managed bundles
✅ Auto-configuration — sensible defaults, override when needed
✅ Externalized configuration — application.yml, env vars, CLI args
✅ Actuator — production monitoring out of the box
✅ No XML — annotation-based, convention over configuration
```

### 2. Auto-Configuration — How It Works

```java
@SpringBootApplication   // = @Configuration + @EnableAutoConfiguration + @ComponentScan

// @EnableAutoConfiguration tells Spring Boot to:
// 1. Scan META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
//    (formerly spring.factories)
// 2. Each auto-config class has @Conditional* annotations
// 3. Only activates if conditions are met

// Example: DataSourceAutoConfiguration
@AutoConfiguration
@ConditionalOnClass(DataSource.class)                   // only if JDBC driver on classpath
@ConditionalOnMissingBean(DataSource.class)             // only if YOU haven't defined one
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}

// Key @Conditional annotations:
// @ConditionalOnClass        — class is on classpath
// @ConditionalOnMissingBean  — no user-defined bean of this type
// @ConditionalOnProperty     — specific property is set
// @ConditionalOnWebApplication — running as a web app
```

💡 **Key insight:** Auto-configuration always defers to your beans. If you define a `DataSource` bean, Spring Boot's auto-configured one backs off (`@ConditionalOnMissingBean`).

### 3. Starters — Dependency Bundles

```xml
<!-- ONE dependency pulls in everything you need -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- Brings: Spring MVC, embedded Tomcat, Jackson, validation -->
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <!-- Brings: Hibernate, Spring Data JPA, HikariCP connection pool -->
</dependency>

<!-- Common starters:
starter-web          → REST APIs
starter-data-jpa     → database with JPA
starter-security     → authentication/authorization
starter-test         → JUnit 5, Mockito, Spring Test
starter-actuator     → monitoring endpoints
starter-validation   → Bean Validation (JSR 380)
-->
```

### 4. @Transactional — Proxy Mechanism Deep Dive

```java
@Service
public class TransferService {
    
    @Transactional     // Spring wraps this class in a CGLIB proxy
    public void transfer(String fromId, String toId, BigDecimal amount) {
        Account from = repo.findById(fromId).orElseThrow();
        Account to = repo.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        // If method completes normally → proxy COMMITS
        // If RuntimeException thrown → proxy ROLLS BACK
        // If checked exception thrown → proxy COMMITS (surprising!)
    }
}

// WHAT ACTUALLY HAPPENS (the proxy):
// 1. Caller calls transferService.transfer(...)
// 2. This goes to the PROXY, not the real object
// 3. Proxy: begin transaction
// 4. Proxy: delegate to real TransferService.transfer()
// 5. If success: proxy commits
// 6. If RuntimeException: proxy rolls back
// 7. If checked exception: proxy COMMITS (unless rollbackFor specified)
```

**Critical gotchas:**

```java
// GOTCHA 1: Self-invocation bypasses the proxy
@Service
public class OrderService {
    @Transactional
    public void processOrder(Order o) { /* transactional */ }
    
    public void batchProcess(List<Order> orders) {
        for (Order o : orders) {
            this.processOrder(o);    // ❌ Direct call — bypasses proxy!
            // No transaction is started because `this` is the real object, not the proxy
        }
    }
}
// Fix: inject self, use TransactionTemplate, or restructure

// GOTCHA 2: Rollback rules
@Transactional                                    // rolls back on RuntimeException ONLY
@Transactional(rollbackFor = Exception.class)     // rolls back on ALL exceptions
@Transactional(noRollbackFor = BusinessException.class)  // don't roll back on specific type

// GOTCHA 3: @Transactional on private methods — does nothing (proxy can't intercept)
```

**Propagation:**
```java
@Transactional(propagation = Propagation.REQUIRED)      // DEFAULT: join existing or create new
@Transactional(propagation = Propagation.REQUIRES_NEW)   // always create new (suspend current)
@Transactional(propagation = Propagation.SUPPORTS)       // use existing if present, none otherwise
@Transactional(propagation = Propagation.MANDATORY)      // MUST have existing (throws if none)
@Transactional(propagation = Propagation.NOT_SUPPORTED)  // suspend current
@Transactional(propagation = Propagation.NEVER)          // throws if tx exists

// Banking example:
@Transactional
public void transfer(String from, String to, BigDecimal amount) {
    debit(from, amount);      // runs in SAME transaction (REQUIRED)
    credit(to, amount);       // runs in SAME transaction
    auditLog(from, to, amount);  // REQUIRES_NEW → audit saved even if transfer rolls back
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(String from, String to, BigDecimal amount) { /* ... */ }
```

### 5. @Async — Asynchronous Execution

```java
@EnableAsync                                // required on @Configuration class
@Configuration
public class AsyncConfig {
    @Bean
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {
    @Async("asyncExecutor")                  // runs on configured thread pool
    public CompletableFuture<String> sendEmail(String to, String body) {
        // runs on a different thread
        emailClient.send(to, body);
        return CompletableFuture.completedFuture("sent");
    }
}

// ⚠️ THREE CAVEATS:
// 1. Self-invocation doesn't work (proxy-based — same as @Transactional)
// 2. Default executor = SimpleAsyncTaskExecutor (creates new thread PER CALL — no pool!)
//    Always configure ThreadPoolTaskExecutor for production
// 3. Exceptions in void @Async methods are SILENTLY SWALLOWED
//    Fix: implement AsyncUncaughtExceptionHandler or return CompletableFuture
```

### 6. Spring Boot Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, env, beans   # expose specific endpoints
  endpoint:
    health:
      show-details: always   # show component health details
```

```java
// Custom health indicator — check downstream dependencies
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        if (isDatabaseReachable()) {
            return Health.up()
                .withDetail("latency", "5ms")
                .build();
        }
        return Health.down()
            .withDetail("error", "Connection refused")
            .build();
    }
}

// Key endpoints:
// /actuator/health     → UP/DOWN status + component health
// /actuator/metrics    → JVM, HTTP, custom metrics (integrates with Micrometer)
// /actuator/env        → all configuration properties (MASK secrets!)
// /actuator/beans      → all beans in the container
// /actuator/threaddump → thread dump (debugging)
// /actuator/heapdump   → heap dump (download .hprof)
// ⚠️ Secure these in production — don't expose env/beans to the internet
```

### 7. Property Binding & Precedence

```
HIGHEST PRIORITY (wins)
  1. Command-line args:            --server.port=9090
  2. SPRING_APPLICATION_JSON
  3. OS environment variables:     SERVER_PORT=9090
  4. application-{profile}.yml:    application-prod.yml
  5. application.yml
  6. @PropertySource
  7. Default properties
LOWEST PRIORITY
```

### 8. Spring Boot Test Slices (Quick Reference)

```java
@SpringBootTest                       // Full context — slow, use for integration tests
@WebMvcTest(OrderController.class)    // Web layer only — fast, mock services
@DataJpaTest                          // JPA layer only — embedded DB, auto-rollback
@WebFluxTest                          // Reactive web layer
@JsonTest                             // Jackson serialisation/deserialisation
@JdbcTest                             // JDBC layer only

@MockBean                            // replace real bean with Mockito mock in context
@SpyBean                             // wrap real bean with Mockito spy
@ActiveProfiles("test")              // load test profile configuration
```

---

## Rapid-Fire Q&A

### Q1: What is Spring Boot and how is it different from Spring?
**A:** Spring Boot = opinionated auto-configuration on top of Spring Framework. Embedded server (Tomcat), no XML, starter dependencies, sensible defaults. Spring is the core framework; Spring Boot makes it production-ready out of the box.

### Q2: How does auto-configuration work?
**A:** `@EnableAutoConfiguration` (included in `@SpringBootApplication`) scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (formerly `spring.factories`). Each auto-config class has `@Conditional*` annotations — only activates if conditions are met (class on classpath, bean not already defined, property set).

### Q3: What are starters?
**A:** Curated dependency bundles. `spring-boot-starter-web` pulls in Spring MVC, embedded Tomcat, Jackson. `spring-boot-starter-data-jpa` pulls in Hibernate, Spring Data JPA. You declare the starter; it manages transitive dependencies.

### Q4: What's `@Transactional` and how does it work?
**A:** Marks a method to run in a database transaction. Spring creates a CGLIB proxy around the bean. Proxy starts a transaction before the method, commits on success, rolls back on unchecked exception. **Gotcha:** self-invocation bypasses the proxy — `this.otherMethod()` won't have transactional behaviour. Solution: inject self or use `TransactionTemplate`.

### Q5: `@Transactional` rollback rules?
**A:** Default: rolls back on unchecked (`RuntimeException`) and `Error`. Does NOT roll back on checked exceptions. Override: `@Transactional(rollbackFor = Exception.class)`. `noRollbackFor` for specific exceptions.

### Q6: What's `@Async` and its caveats?
**A:** Runs the method on a separate thread, returns `CompletableFuture<T>` or `void`. Requires `@EnableAsync`. Caveats: (1) self-invocation doesn't work (proxy-based). (2) Default uses `SimpleAsyncTaskExecutor` — creates a new thread per call. Configure `ThreadPoolTaskExecutor` for production. (3) Uncaught exceptions are silently swallowed unless you configure `AsyncUncaughtExceptionHandler`.

### Q7: Spring Boot Actuator — what does it provide?
**A:** Production monitoring endpoints: `/health`, `/info`, `/metrics`, `/env`, `/beans`, `/threaddump`, `/heapdump`. Secure them in production. Custom health indicators for downstream dependencies. Integrates with Micrometer for Prometheus/Grafana.

### Q8: How do you handle errors globally in Spring Boot?
**A:** `@ControllerAdvice` + `@ExceptionHandler(SpecificException.class)`. Returns a consistent error response. For REST: `@RestControllerAdvice`. For 404s: `spring.mvc.throw-exception-if-no-handler-found=true`.

### Q9: Spring Boot scheduling?
**A:** `@EnableScheduling` + `@Scheduled(fixedRate=5000)` or `@Scheduled(cron="0 0 * * * *")`. Runs in a single-threaded scheduler by default — configure `ThreadPoolTaskScheduler` for concurrent scheduled tasks.

### Q10: What's `application.yml` property precedence?
**A:** (highest to lowest): Command-line args → env variables → `application-{profile}.yml` → `application.yml` → `@PropertySource` → defaults. Profile-specific properties override base properties.

### Q11: How do you run integration tests in Spring Boot?
**A:** `@SpringBootTest` loads the full application context. `@MockBean` replaces beans with mocks. `@WebMvcTest` loads only the web layer (fast). `@DataJpaTest` loads only JPA components with embedded DB. Use `@ActiveProfiles("test")` for test configuration.

---

## Can you answer these cold?

- [ ] Auto-configuration — how `@Conditional*` works
- [ ] `@Transactional` — proxy mechanism, self-invocation trap, rollback rules
- [ ] `@Async` — three caveats (proxy, executor, exception handling)
- [ ] Actuator endpoints — `/health`, `/metrics`, custom health indicators
- [ ] Property precedence — command line > env > profile yml > base yml
- [ ] Test slices — `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`

[← Back to Index](./00_INDEX.md)
