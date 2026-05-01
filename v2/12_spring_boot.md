# 12 — Spring Boot

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

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
