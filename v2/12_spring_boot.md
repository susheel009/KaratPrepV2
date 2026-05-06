# 12 — Spring Boot

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/12_spring_boot_doubts.md) · [← Prev: 11 Spring Core](./11_spring_core.md) · [Next: 13 Spring REST →](./13_spring_rest.md)
>
> **Priority:** 🟡 High · **Related topics:** [11 Spring Core](./11_spring_core.md) · [13 Spring REST](./13_spring_rest.md) · [14 Spring AOP & Data](./14_spring_aop_data.md) · [01 Concurrency (`@Async`)](./01_concurrency.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — making Spring usable in 5 minutes, not 5 days
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — auto-config flow
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — running app in 15 lines
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Auto-configuration](#41-auto-configuration--how-it-actually-works)
   - [4.2 Starters](#42-starters--curated-dependency-bundles)
   - [4.3 Externalised config](#43-externalised-config--applicationyml--profiles)
   - [4.4 Property precedence](#44-property-precedence)
   - [4.5 @Transactional](#45-transactional--the-proxy-mechanism)
   - [4.6 @Async](#46-async--background-execution)
   - [4.7 @Scheduled](#47-scheduled--periodic-jobs)
   - [4.8 Actuator](#48-actuator--production-monitoring)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 @Transactional propagation](#51-transactional-propagation)
   - [5.2 @Transactional rollback rules](#52-transactional-rollback-rules)
   - [5.3 Self-invocation traps](#53-self-invocation-traps-transactional-async-cacheable)
   - [5.4 Auto-configuration class report](#54-debugging-auto-configuration-the-conditions-report)
   - [5.5 Banner / startup events](#55-startup-event-lifecycle)
   - [5.6 Test slices recap](#56-test-slices-recap)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Plain Spring (the framework underlying Spring Boot, covered in [11](./11_spring_core.md)) gives you the IoC container and a wealth of capabilities. But to *start* a Spring application before Spring Boot, you'd:

1. Pick versions for ~30 transitively-related libraries (Spring core, MVC, security, data, validation, Jackson, embedded server, logging, …).
2. Write XML or `@Configuration` classes to wire your servlet container, register dispatchers, configure JSON, set up the data source, …
3. Build a `WAR`, deploy it to an external Tomcat or JBoss, configure that server's connector, port, JVM options.
4. Repeat the whole exercise per environment (dev, staging, prod).

Most of this is boilerplate. Most teams reinvent the same defaults. Spring Boot is the response: **package the boring decisions, ship them as opinionated defaults, let you override only what you actually need**.

Three core moves:

- **Starters** — one Maven/Gradle dependency (`spring-boot-starter-web`) pulls in a curated, version-aligned set of libraries that just work together.
- **Auto-configuration** — at startup, Boot inspects the classpath, the existing beans, and properties; for every "feature" (JDBC, JPA, Kafka, security, …) it enables a sensible default config *only if you haven't already provided one*. Add `H2` to the classpath, get an embedded `DataSource`. Add Jackson, get a configured `ObjectMapper`. Define your own `DataSource` bean? Boot's gets out of the way.
- **Embedded server** — your app is a runnable JAR with Tomcat/Jetty/Undertow inside. `java -jar app.jar` starts it. No external server. No WAR deployment.

> Once you internalise "Boot is Spring + curated defaults + an embedded server", every Boot annotation (`@SpringBootApplication`, `@EnableAutoConfiguration`, `@ConditionalOnClass`, `@ConfigurationProperties`) maps to one of those three moves.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

`SpringApplication.run(...)` starts. Watch the auto-config decision flow for one feature — the `DataSource`:

| Step | What happens |
|---|---|
| 1 | Boot's auto-config scanner reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Hundreds of `@AutoConfiguration` classes are listed. |
| 2 | For each candidate (e.g. `DataSourceAutoConfiguration`), Boot evaluates its `@Conditional*` annotations. |
| 3 | `@ConditionalOnClass(DataSource.class)` — is there a JDBC driver on the classpath? If yes, proceed; if no, skip this auto-config. |
| 4 | `@ConditionalOnMissingBean(DataSource.class)` — has the user already defined a `DataSource` bean? If yes, skip (yield to the user); if no, proceed. |
| 5 | `@EnableConfigurationProperties(DataSourceProperties.class)` — bind `spring.datasource.*` from `application.yml` into a typed object. |
| 6 | The `@Bean DataSource dataSource(...)` method runs, building a HikariCP-backed pool from the resolved properties. |
| 7 | The bean joins the container. Anything depending on `DataSource` (a `JpaRepository`, a `JdbcTemplate`) gets wired. |

Two takeaways the walkthrough makes obvious:

1. **Auto-configuration always defers to you.** If you define a bean, Boot's matching auto-config bows out (`@ConditionalOnMissingBean`). You're never fighting the framework.
2. **The classpath drives everything.** Add `spring-boot-starter-data-jpa` → JPA configures itself. Remove it → silently disappears. Adding/removing dependencies *is* configuration in Spring Boot.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
@SpringBootApplication                                   // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);          // boots context, starts embedded Tomcat
    }
}

@RestController                                          // = @Controller + @ResponseBody on every method
@RequestMapping("/api")
class HelloController {
    @GetMapping("/hello/{name}")
    public Map<String, String> hello(@PathVariable String name) {
        return Map.of("message", "Hello, " + name);      // returned as JSON automatically (Jackson)
    }
}
```

```yaml
# application.yml
server:
  port: 8080
spring:
  application:
    name: hello-app
```

What just happened: 12 lines of Java, an embedded Tomcat boots on port 8080, the `/api/hello/{name}` endpoint returns JSON. No XML, no WAR, no external server. Add `spring-boot-starter-data-jpa` and one annotation on a `Repository` interface and you have working JPA persistence — same idea, scaled.

---

## 4. Build Up — Practical Patterns

### 4.1 Auto-configuration — how it actually works

```java
// Conceptually, every auto-config class looks like this:
@AutoConfiguration                                  // marker
@ConditionalOnClass(DataSource.class)               // only if JDBC on classpath
@ConditionalOnMissingBean(DataSource.class)         // only if user hasn't supplied one
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

Common `@Conditional*` annotations:

| Annotation | Meaning |
|---|---|
| `@ConditionalOnClass(X.class)` | X is on the classpath |
| `@ConditionalOnMissingClass("X")` | X is **not** on the classpath |
| `@ConditionalOnBean(X.class)` | A bean of type X already exists in the context |
| `@ConditionalOnMissingBean(X.class)` | No bean of type X exists yet |
| `@ConditionalOnProperty("prop", havingValue = "true")` | Property has a specific value |
| `@ConditionalOnWebApplication` | Running as a web app |
| `@ConditionalOnExpression("#{...}")` | SpEL expression evaluates true |

The order of evaluation respects `@AutoConfigureBefore` / `@AutoConfigureAfter` so user-supplied config wins.

> 💡 To see what auto-config did or didn't activate: run with `--debug` or the `/actuator/conditions` endpoint. Both list every condition and its outcome. Indispensable for debugging "why isn't my bean wired?".

### 4.2 Starters — curated dependency bundles

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <!-- Brings: Spring MVC, embedded Tomcat, Jackson, validation, logging — all version-aligned -->
</dependency>
```

| Starter | Includes |
|---|---|
| `spring-boot-starter-web` | Spring MVC, embedded Tomcat, Jackson, Bean Validation |
| `spring-boot-starter-webflux` | Reactive WebFlux, embedded Netty, reactive stack |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, HikariCP |
| `spring-boot-starter-data-redis` | Lettuce/Jedis client, Spring Data Redis |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Spring Test |
| `spring-boot-starter-actuator` | Health/metrics/info endpoints |
| `spring-boot-starter-validation` | Bean Validation (Jakarta Validation) |

### 4.3 Externalised config — `application.yml` + profiles

```yaml
# application.yml — defaults
server:
  port: 8080
spring:
  application:
    name: trading-api
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa

logging:
  level:
    com.example: INFO
```

```yaml
# application-prod.yml — overrides when prod profile is active
server:
  port: 80
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/trading
    username: ${DB_USER}
    password: ${DB_PASS}                # from env var
logging:
  level:
    com.example: WARN
```

Activate a profile with `spring.profiles.active=prod` (env var, CLI arg, or in YAML). See [11 §4.7](./11_spring_core.md#47-qualifier-primary-and-profiles) for `@Profile`-annotated bean variants.

### 4.4 Property precedence

When the same property is set in multiple places, the **highest** wins:

```
HIGHEST (wins)
  1. Command-line args:                   --server.port=9090
  2. SPRING_APPLICATION_JSON
  3. ServletConfig / ServletContext init params
  4. JNDI from java:comp/env
  5. Java System properties (-Dprop=val)
  6. OS environment variables:            SERVER_PORT=9090
  7. application-{profile}.yml / .properties
  8. application.yml / .properties (in src/main/resources)
  9. application.yml inside JARs
 10. @PropertySource on @Configuration classes
 11. SpringApplication.setDefaultProperties
LOWEST
```

The two everyday rules: **environment variables override YAML**, and **profile-specific YAML overrides base YAML**. This lets a single image run anywhere, configured by env vars in the orchestrator (Kubernetes, Docker Compose, ECS).

### 4.5 `@Transactional` — the proxy mechanism

```java
@Service
public class TransferService {
    @Transactional
    public void transfer(String fromId, String toId, BigDecimal amount) {
        Account from = repo.findById(fromId).orElseThrow();
        Account to   = repo.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        // If method completes normally → proxy COMMITS.
        // If RuntimeException thrown → proxy ROLLS BACK.
        // If checked exception thrown → proxy COMMITS (unless rollbackFor specified).
    }
}
```

The mechanism — `@Transactional` works because Spring wraps your bean in a CGLIB or JDK proxy at startup ([08 §4.6](./08_design_patterns.md#46-proxy--control-access-or-add-cross-cutting-behaviour)). The proxy:

```
client                 ProxyTransferService                      RealTransferService
   │                          │                                          │
   │  transfer(...)           │                                          │
   ├─────────────────────────▶│ begin tx                                 │
   │                          ├─────────────────────────────────────────▶│ method body runs
   │                          │                                          │
   │                          │  on success: commit                      │
   │                          │  on RuntimeException: rollback            │
   │                          │                                          │
   │◀─────────────────────────┤ return / rethrow                          │
```

> ⚠ **Self-invocation skips the proxy.** Calls through `this.method()` go directly to the real instance — the proxy isn't in the path, so no transaction starts. See [§5.3](#53-self-invocation-traps-transactional-async-cacheable).

### 4.6 `@Async` — background execution

```java
@SpringBootApplication
@EnableAsync
public class App { ... }

@Configuration
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        var ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(5);
        ex.setMaxPoolSize(20);
        ex.setQueueCapacity(100);
        ex.setThreadNamePrefix("async-");
        ex.initialize();
        return ex;
    }
}

@Service
public class NotificationService {
    @Async("taskExecutor")
    public CompletableFuture<String> send(String to, String body) {
        emailClient.send(to, body);
        return CompletableFuture.completedFuture("sent");
    }
}
```

Three caveats:

1. **Self-invocation doesn't work** (proxy bypass — same as `@Transactional`).
2. **Default executor is `SimpleAsyncTaskExecutor`** — creates a new thread *per call*, no pooling, no queue. **Always configure `ThreadPoolTaskExecutor`** in production.
3. **Exceptions in `void @Async` methods are silently swallowed** unless you configure `AsyncUncaughtExceptionHandler`. Returning `CompletableFuture<T>` propagates the exception to the caller.

### 4.7 `@Scheduled` — periodic jobs

```java
@SpringBootApplication
@EnableScheduling
public class App { ... }

@Service
public class CleanupJob {
    @Scheduled(fixedRate = 60_000)             // every 60 seconds (next start measured from previous start)
    public void purgeStaleSessions() { /* ... */ }

    @Scheduled(fixedDelay = 60_000)            // 60 seconds AFTER previous finish
    public void slowJob() { /* ... */ }

    @Scheduled(cron = "0 0 2 * * *", zone = "America/New_York")    // 02:00 NY every day
    public void nightlyReport() { /* ... */ }
}
```

By default `@Scheduled` runs on a **single-threaded** scheduler — long jobs delay the next tick. Configure a `ThreadPoolTaskScheduler` if you have multiple scheduled methods that mustn't block each other.

### 4.8 Actuator — production monitoring

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when_authorized
```

| Endpoint | What |
|---|---|
| `/actuator/health` | UP/DOWN + per-component (DB, disk, mq) status |
| `/actuator/info` | Build / git info, custom info |
| `/actuator/metrics` | JVM, HTTP, custom metrics (Micrometer) |
| `/actuator/prometheus` | Metrics in Prometheus scrape format |
| `/actuator/env` | All resolved properties — **mask secrets** |
| `/actuator/beans` | All beans in the context |
| `/actuator/threaddump` | Thread dump |
| `/actuator/heapdump` | Downloads `.hprof` |
| `/actuator/conditions` | Auto-configuration evaluation report |

```java
// Custom health indicator
@Component
public class CacheHealthIndicator implements HealthIndicator {
    public Health health() {
        if (cache.isReachable()) return Health.up().withDetail("size", cache.size()).build();
        return Health.down().withDetail("error", "unreachable").build();
    }
}
```

> ⚠ **Secure Actuator in production.** `/env`, `/beans`, `/heapdump`, `/threaddump` leak sensitive information. Expose only `/health` and `/info` to the public; gate the rest behind authentication.

---

## 5. Going Deep — Interview-Level Material

### 5.1 `@Transactional` propagation

| Propagation | What happens |
|---|---|
| `REQUIRED` (default) | Join the existing tx; create one if none. |
| `REQUIRES_NEW` | Suspend the existing tx, run a brand-new tx independently, then resume. |
| `NESTED` | Run inside a savepoint within the existing tx (rollback to savepoint, not the whole tx). |
| `MANDATORY` | Must have an existing tx — throws `IllegalTransactionStateException` if none. |
| `SUPPORTS` | Use existing tx if present; run non-transactionally otherwise. |
| `NOT_SUPPORTED` | Suspend the existing tx; run non-transactionally. |
| `NEVER` | Throws if a tx exists; runs non-transactionally otherwise. |

Common pattern — audit log that must persist *even if* the business operation rolls back:

```java
@Transactional                                         // REQUIRED — the main tx
public void transfer(String from, String to, BigDecimal amount) {
    debit(from, amount);
    credit(to, amount);
    auditService.record(from, to, amount);             // runs in REQUIRES_NEW — separate tx
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(String from, String to, BigDecimal amount) { /* ... */ }
}
```

If `transfer` throws after calling `audit.record`, the audit row stays — the audit was committed in its own transaction.

### 5.2 `@Transactional` rollback rules

```java
@Transactional                                          // rolls back on RuntimeException + Error
@Transactional(rollbackFor = Exception.class)            // rolls back on any Exception (including checked)
@Transactional(noRollbackFor = BusinessException.class)  // commit even when this exception escapes
```

The default — rollback only on unchecked exceptions — is a frequent source of surprise. A method that throws a custom checked exception will **commit** unless you set `rollbackFor`. Modern Spring code mostly uses unchecked exceptions, dodging the question; if you have legacy checked exceptions, set `rollbackFor` explicitly.

### 5.3 Self-invocation traps (`@Transactional`, `@Async`, `@Cacheable`)

All three are implemented via proxies. Calls that don't go through the proxy don't get the cross-cutting behaviour:

```java
@Service
public class OrderService {
    @Transactional public void placeOrder(Order o) { /* ... */ }

    public void batchProcess(List<Order> orders) {
        for (Order o : orders) {
            this.placeOrder(o);             // ❌ direct call, NO transaction
        }
    }
}
```

Three fixes:

```java
// 1. Inject self — the injected reference IS the proxy
@Service
public class OrderService {
    @Lazy @Autowired private OrderService self;
    public void batchProcess(List<Order> orders) {
        for (Order o : orders) self.placeOrder(o);     // through proxy ✅
    }
}

// 2. Use TransactionTemplate explicitly (no annotation)
@Service
public class OrderService {
    private final TransactionTemplate txTemplate;
    public void batchProcess(List<Order> orders) {
        for (Order o : orders) {
            txTemplate.executeWithoutResult(status -> placeOrderInternal(o));
        }
    }
}

// 3. Restructure — put the loop in another bean that calls placeOrder externally
```

### 5.4 Debugging auto-configuration: the conditions report

When something isn't wired the way you expect, two tools tell you why:

- **`--debug` on the command line** — prints a "CONDITIONS EVALUATION REPORT" with every auto-config class, the conditions that matched, and the conditions that didn't.
- **`/actuator/conditions`** — same report, exposed as JSON at runtime.

Typical findings:

- "Did not match: `@ConditionalOnMissingBean`. Found bean of type X." → an auto-config didn't activate because *you* defined the bean. Often intentional.
- "Did not match: `@ConditionalOnClass`." → a starter you expected isn't on the classpath. Check Maven dependencies.
- "Matched: …" with the wrong implementation → an unexpected starter is on the classpath bringing its own auto-config. Use `mvn dependency:tree` to find the source.

### 5.5 Startup event lifecycle

Boot publishes a sequence of events you can hook into:

```
ApplicationStartingEvent
    │
    ▼
ApplicationEnvironmentPreparedEvent          ← Environment ready; bean defs not yet loaded
    │
    ▼
ApplicationContextInitializedEvent
    │
    ▼
ApplicationPreparedEvent                     ← Context refreshed; beans not yet started
    │
    ▼
ApplicationStartedEvent                      ← All non-lazy beans started; web server may not yet be ready
    │
    ▼
ApplicationReadyEvent                        ← App is ready to serve traffic
    │
    ▼
ApplicationFailedEvent (only if startup fails)
```

```java
@EventListener
public void onReady(ApplicationReadyEvent e) {
    log.info("App is up and ready to serve");        // hook for "ready" metric, warm caches, etc.
}
```

### 5.6 Test slices recap

Cross-link to [10 Testing](./10_testing.md):

| Slice | Loads | Use |
|---|---|---|
| `@SpringBootTest` | Full context (slow) | Integration tests crossing many layers |
| `@WebMvcTest(Controller.class)` | Web layer + controllers, no JPA | Controller tests |
| `@DataJpaTest` | JPA + embedded DB, auto-rollback | Repository tests |
| `@JsonTest` | Jackson | JSON serialization tests |
| `@WebFluxTest` | Reactive web layer | WebFlux controller tests |
| `@RestClientTest` | RestClient/RestTemplate + MockRestServiceServer | HTTP client tests |
| `@JdbcTest` | JDBC + embedded DB | Plain JDBC code |

Use `@MockBean` to replace a real bean with a mock inside the loaded context. Use `@SpyBean` to wrap with a spy.

---

## 6. Memory Aids

### Decision tree: "where should I put this configuration?"

```
Is it environment-specific (different per dev / staging / prod)?
├── Yes → application-{profile}.yml + spring.profiles.active=...
└── No
    ├── Is it a single value? → @Value("${prop}") on the field/parameter (small).
    └── A group of related properties? → @ConfigurationProperties record.

Is it sensitive (password, API key)?
├── Yes → environment variable, NOT YAML committed to git.
└── No  → YAML is fine.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Spring Boot vs Spring?" | Curated defaults + auto-config + embedded server | "Spring is the framework; Boot is opinionated tooling on top." |
| "How does auto-config work?" | `@Conditional*` on class / bean / property | "Activates a default config only when prerequisites match and you haven't supplied your own." |
| "Why does my bean not get the transaction?" | Self-invocation through `this.` | "The proxy isn't in the path. Inject self or restructure." |
| "@Transactional rollback default?" | RuntimeException + Error only | "Checked exceptions DO NOT roll back unless rollbackFor is set." |
| "@Async default executor?" | `SimpleAsyncTaskExecutor` (no pool) | "Always configure `ThreadPoolTaskExecutor` in production." |
| "Property precedence?" | CLI > env > profile YAML > base YAML | "Highest wins. Env vars override YAML — that's the deployment story." |
| "Actuator security?" | Treat as sensitive | "Expose only `/health` + `/info` publicly; auth on the rest." |
| "How do I debug 'wrong/no bean'?" | `--debug` or `/actuator/conditions` | "Conditions report shows every auto-config decision." |

### Three anchor pictures

1. **Auto-config defers to you.** Define the bean → Boot's version backs off.
2. **Cross-cutting = proxy.** `@Transactional`, `@Async`, `@Cacheable` all rely on the proxy. Self-invocation skips it.
3. **YAML for static defaults; env vars for everything sensitive or env-specific.**

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: What is Spring Boot and how is it different from Spring?
**A:** Spring Boot is opinionated auto-configuration on top of Spring. Embedded server (Tomcat/Jetty/Undertow), starter dependencies, sensible defaults, externalised config. Spring is the core framework; Boot makes it production-ready out of the box without XML or external servers.

### Q2: How does auto-configuration work?
**A:** `@EnableAutoConfiguration` (included in `@SpringBootApplication`) scans `META-INF/spring/...AutoConfiguration.imports`. Each auto-config class has `@Conditional*` annotations — only activates when conditions match (class on classpath, bean not already defined, property set, etc.). User beans always win via `@ConditionalOnMissingBean`.

### Q3: What are starters?
**A:** Curated dependency bundles. `spring-boot-starter-web` brings Spring MVC + embedded Tomcat + Jackson + Validation, version-aligned. You declare the starter; it manages all transitive dependencies.

### Q4: What does `@SpringBootApplication` include?
**A:** `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` (current package + subpackages).

### Q5: What's `@Transactional` and how does it work?
**A:** Marks a method to run in a transaction. Spring wraps the bean in a CGLIB/JDK proxy. The proxy starts a transaction before the method, commits on normal return, rolls back on `RuntimeException` (default). **Gotcha:** self-invocation (`this.method()`) bypasses the proxy — no transaction. Fix: inject self via `@Lazy`, use `TransactionTemplate`, or restructure.

### Q6: `@Transactional` rollback rules?
**A:** Default rolls back on `RuntimeException` and `Error`. Does **not** roll back on checked exceptions. Override with `@Transactional(rollbackFor = Exception.class)` or `noRollbackFor`.

### Q7: `@Transactional` propagation — when use each?
**A:** `REQUIRED` (default — join or create), `REQUIRES_NEW` (always new, suspend existing — useful for audit logs that must persist even if main tx rolls back), `NESTED` (savepoint), `MANDATORY` (must have one), `SUPPORTS` / `NOT_SUPPORTED` / `NEVER` (rare).

### Q8: What's `@Async` and its caveats?
**A:** Runs the method on another thread, returns `CompletableFuture<T>` or `void`. Requires `@EnableAsync`. Three caveats: (1) self-invocation bypasses the proxy (same as `@Transactional`), (2) default executor `SimpleAsyncTaskExecutor` creates a thread per call — configure `ThreadPoolTaskExecutor` in production, (3) exceptions in void `@Async` methods are silently swallowed unless you set an `AsyncUncaughtExceptionHandler`.

### Q9: What's `@Scheduled`?
**A:** Periodic execution. `fixedRate` (every N ms from start), `fixedDelay` (N ms after finish), `cron = "..."`. Requires `@EnableScheduling`. Default scheduler is single-threaded — configure `ThreadPoolTaskScheduler` if you need concurrent scheduled methods.

### Q10: Spring Boot Actuator — what does it provide?
**A:** Production monitoring endpoints: `/health`, `/info`, `/metrics`, `/prometheus`, `/env`, `/beans`, `/threaddump`, `/heapdump`, `/conditions`. Custom `HealthIndicator` for downstream dependencies. Integrates with Micrometer for Prometheus/Grafana. **Secure in production** — don't expose `/env`, `/beans`, `/heapdump` publicly.

### Q11: How do you handle errors globally in Spring Boot?
**A:** `@RestControllerAdvice` class with `@ExceptionHandler` methods — converts exception types into HTTP responses. Returns a consistent error body. See [13 Spring REST §5](./13_spring_rest.md).

### Q12: Property precedence?
**A:** Highest to lowest: command-line args → `SPRING_APPLICATION_JSON` → JNDI → system properties → env variables → `application-{profile}.yml` → `application.yml` → `@PropertySource` → `setDefaultProperties`. Profile-specific YAML overrides base; env vars override YAML.

### Q13: How do you debug "my bean isn't wired" or "wrong bean wired"?
**A:** Run with `--debug` to print the Conditions Evaluation Report, or hit `/actuator/conditions`. Each auto-config class lists its matched and unmatched conditions. Often reveals "did not match — found existing bean" (your bean is overriding) or "did not match — class not present" (missing starter).

### Q14: Test slices in Spring Boot?
**A:** `@SpringBootTest` (full context, slow), `@WebMvcTest(Controller.class)` (web layer, fast), `@DataJpaTest` (JPA + embedded DB), `@JsonTest` (Jackson), `@WebFluxTest`, `@RestClientTest`, `@JdbcTest`. `@MockBean` replaces a real bean with a Mockito mock inside the context.

### Q15: How does `@ConfigurationProperties` differ from `@Value`?
**A:** `@Value("${app.x}")` — single property, no validation, no grouping. `@ConfigurationProperties(prefix = "app")` — binds a YAML/properties prefix to a record/POJO, supports `Duration` / `DataSize` / list parsing, and `@Validated` for Bean Validation. Prefer `@ConfigurationProperties` for non-trivial config.

### Q16: How does Spring Boot package an app?
**A:** Single executable JAR. `mvn package` produces `app-X.Y.Z.jar`. Run with `java -jar app.jar`. Inside: your code, all dependency JARs nested, embedded server. No external Tomcat needed.

---

### Key Code Patterns

**`@SpringBootApplication` entry point**
```java
@SpringBootApplication
public class App {
    public static void main(String[] args) { SpringApplication.run(App.class, args); }
}
```

**`@Transactional` with explicit rollback**
```java
@Transactional(rollbackFor = Exception.class)
public void process(Job j) throws JobException { ... }
```

**Configured async executor**
```java
@Bean(name = "taskExecutor")
public Executor taskExecutor() {
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(5); ex.setMaxPoolSize(20); ex.setQueueCapacity(100);
    ex.setThreadNamePrefix("async-"); ex.initialize();
    return ex;
}
```

**Custom health indicator**
```java
@Component
public class DownstreamHealth implements HealthIndicator {
    public Health health() {
        return ping() ? Health.up().build() : Health.down().withDetail("err", "unreachable").build();
    }
}
```

**`@ConfigurationProperties` record**
```java
@ConfigurationProperties("app.email")
public record EmailProps(@NotBlank String host, int port, Duration timeout) {}
```

---

## 8. Self-Test

**Easy**
- [ ] What does `@SpringBootApplication` combine?
- [ ] How does Spring Boot package an app?
- [ ] What does `--debug` give you at startup?
- [ ] What's a starter?

**Medium**
- [ ] Walk through what happens at startup for one auto-config class.
- [ ] Why does `@Transactional` need a proxy? Show one self-invocation trap.
- [ ] What's the difference between `fixedRate` and `fixedDelay` on `@Scheduled`?
- [ ] When should you set `@Transactional(rollbackFor = ...)`?

**Hard**
- [ ] Trace a transaction with `REQUIRES_NEW` for an audit log inside a `REQUIRED` outer tx that throws.
- [ ] Why are `@Async` exceptions on void methods silently swallowed? How do you fix it?
- [ ] How would you secure Actuator endpoints in production?
- [ ] Read a `/actuator/conditions` snippet and explain what it tells you.
- [ ] Describe the lifecycle from `ApplicationStartingEvent` to `ApplicationReadyEvent`.

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Auto-configuration** | Boot's mechanism of activating sensible defaults based on the classpath and existing beans. |
| **Starter** | Curated dependency bundle (`spring-boot-starter-web`). |
| **Embedded server** | Tomcat/Jetty/Undertow inside the JAR — `java -jar` starts a web app. |
| **`@SpringBootApplication`** | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. |
| **`@Conditional*`** | Annotations gating bean creation on classpath / bean / property. |
| **Profile** | Named config set activated via `spring.profiles.active`. |
| **`@ConfigurationProperties`** | Type-safe binding of a YAML prefix to a record/POJO. |
| **`@Transactional`** | Method runs inside a database transaction. Implemented via proxy. |
| **Propagation** | How a `@Transactional` method behaves relative to existing transactions. |
| **`@Async`** | Method runs on a different thread; returns `CompletableFuture<T>` or void. |
| **`@Scheduled`** | Method runs on a timer (fixedRate, fixedDelay, cron). |
| **Actuator** | Production monitoring endpoints (`/health`, `/metrics`, `/info`). |
| **Self-invocation** | Calling `this.otherMethod()` — bypasses Spring proxy. |
| **`@RestControllerAdvice`** | Class with `@ExceptionHandler` methods for global error handling. |
| **Test slice** | Annotation loading just one Boot layer (`@WebMvcTest`, `@DataJpaTest`). |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/12_spring_boot_doubts.md) · [← Prev: 11 Spring Core](./11_spring_core.md) · [Next: 13 Spring REST →](./13_spring_rest.md)

[↑ Back to top](#12--spring-boot)
