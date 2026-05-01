# 11 — Spring Core & Dependency Injection

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## 🟢 Start Here — Spring & Dependency Injection in Plain English

### What is Spring?

Spring is a **framework** — a toolkit that helps you build Java applications faster. Without Spring, you'd write tons of boilerplate code to wire things together. With Spring, you describe *what you need*, and Spring *provides it for you*.

### What is Dependency Injection (DI)?

Imagine you're a chef. You need ingredients (dependencies). Two approaches:

**Without DI:** You walk to the store, buy ingredients yourself, know every store's address. If the store changes, you have to update all your code.

**With DI:** Someone (the container) **delivers the ingredients to your kitchen**. You just say "I need eggs" and they appear. If the egg supplier changes, you don't care — the delivery person handles it.

```java
// WITHOUT Spring — you create everything yourself
class OrderService {
    OrderRepository repo = new MySQLOrderRepository();  // YOU chose MySQL
    // What if you want to test with a fake database? Hard to swap.
}

// WITH Spring — dependencies are delivered to you
@Service
class OrderService {
    private final OrderRepository repo;  // just declare what you NEED
    
    OrderService(OrderRepository repo) {  // Spring delivers it
        this.repo = repo;
    }
}
// In production: Spring provides MySQLOrderRepository
// In tests: YOU provide a mock — easy to swap!
```

### What's a "bean"?

A **bean** is just an object that Spring manages. You mark your class with `@Service`, `@Component`, or `@Repository`, and Spring automatically creates it and keeps track of it.

```java
@Service          // tells Spring: "manage this class for me"
class OrderService { }

@Repository       // tells Spring: "this is a data-access class"
class OrderRepository { }

// Spring creates one instance of each and keeps them in its "container"
// When OrderService needs OrderRepository, Spring wires them together automatically
```

### Why is this useful?

1. **Easy to test** — swap real database with a mock in one line
2. **Loose coupling** — classes don't know about each other's implementations
3. **Less boilerplate** — Spring handles object creation, lifecycle, configuration

> Key takeaway: Spring = a framework that wires your app together. DI = Spring delivers dependencies to your classes instead of you creating them. Beans = objects managed by Spring. This makes your code easy to test and change.

---

## 📚 Study Material

### 1. IoC (Inversion of Control) — The Core Idea

**Without IoC:** Your code creates its own dependencies.
**With IoC:** A container creates and wires dependencies for you.

```java
// WITHOUT IoC — tight coupling
public class OrderService {
    private final OrderRepository repo = new MySQLOrderRepository();  // YOU create it
    private final EmailService email = new SmtpEmailService();        // YOU create it
    // Can't test without real MySQL and SMTP server
}

// WITH IoC (Spring) — container controls lifecycle
@Service
public class OrderService {
    private final OrderRepository repo;       // container provides it
    private final EmailService email;         // container provides it
    
    public OrderService(OrderRepository repo, EmailService email) {
        this.repo = repo;                     // injected at construction time
        this.email = email;
    }
}
// In tests: pass mocks via constructor. In prod: Spring wires real implementations.
```

### 2. Dependency Injection — Three Types

```java
// 1. CONSTRUCTOR INJECTION (preferred ✅)
@Service
public class TradeService {
    private final TradeRepository repo;         // final — immutable
    private final AuditService audit;           // explicit — can't forget a dependency
    
    public TradeService(TradeRepository repo, AuditService audit) {  // since Spring 4.3: no @Autowired needed for single constructor
        this.repo = repo;
        this.audit = audit;
    }
}
// WHY preferred:
// ✅ Dependencies are explicit (visible in constructor)
// ✅ Fields can be final (immutable, thread-safe)
// ✅ Easy to test (pass mocks via constructor — no Spring needed)
// ✅ Circular dependencies fail fast at startup (good — it's a design smell)

// 2. SETTER INJECTION (for optional dependencies)
@Service
public class ReportService {
    private CacheService cache;               // optional — might not be present
    
    @Autowired(required = false)              // won't fail if no CacheService bean
    public void setCache(CacheService cache) {
        this.cache = cache;
    }
}

// 3. FIELD INJECTION (discouraged ❌)
@Service
public class PaymentService {
    @Autowired                                // hidden dependency
    private PaymentGateway gateway;           // can't be final
    // ❌ Need reflection to inject in tests
    // ❌ Dependencies invisible from outside
    // ❌ Easy to add too many (no constructor pain = no design pressure)
}
```

### 3. Bean Scopes

```java
@Component
@Scope("singleton")       // DEFAULT — one instance per container (shared)
public class AppConfig { }

@Component
@Scope("prototype")       // new instance every time it's requested/injected
public class RequestProcessor { }

@Component
@Scope("request")         // one instance per HTTP request (web only)
public class RequestContext { }

@Component
@Scope("session")         // one instance per HTTP session (web only)
public class UserSession { }
```

**The Prototype-in-Singleton Problem:**
```java
// ❌ PROBLEM: Prototype bean injected into Singleton gets created ONCE
@Service                   // singleton
public class OrderService {
    @Autowired             // injected once at startup — never refreshed!
    private RequestProcessor processor;   // prototype — but you only get ONE instance
}

// ✅ FIX 1: @Lookup method (Spring overrides via CGLIB)
@Service
public abstract class OrderService {
    @Lookup
    abstract RequestProcessor getProcessor();  // Spring returns fresh prototype each call
    
    public void process(Order o) {
        RequestProcessor p = getProcessor();   // new instance each time
    }
}

// ✅ FIX 2: ObjectFactory / Provider
@Service
public class OrderService {
    private final ObjectFactory<RequestProcessor> processorFactory;
    
    public OrderService(ObjectFactory<RequestProcessor> processorFactory) {
        this.processorFactory = processorFactory;
    }
    
    public void process(Order o) {
        RequestProcessor p = processorFactory.getObject();  // new instance each time
    }
}
```

### 4. Bean Lifecycle

```
Container starts
    │
    ▼
Constructor called (instantiation)
    │
    ▼
Dependencies injected (@Autowired / constructor)
    │
    ▼
@PostConstruct method runs
    │  (or InitializingBean.afterPropertiesSet())
    ▼
Bean is READY — application uses it
    │
    ▼
@PreDestroy method runs (on shutdown)
    │  (or DisposableBean.destroy())
    ▼
Bean destroyed
```

```java
@Service
public class CacheService {
    
    @PostConstruct                      // runs AFTER all dependencies injected
    public void init() {
        loadCache();                    // warm up cache at startup
        log.info("Cache initialized");
    }
    
    @PreDestroy                         // runs on container shutdown (graceful)
    public void cleanup() {
        flushCache();                   // persist cache before shutdown
        log.info("Cache flushed");
    }
}

// BeanPostProcessor — cross-cutting initialisation logic
// Applied to ALL beans. Used by Spring internally for:
// - @Autowired processing
// - @Transactional proxy creation
// - @Async proxy creation
```

### 5. Component Scanning & Stereotypes

```java
@SpringBootApplication               // includes @ComponentScan (scans current package + sub-packages)
public class Application { }

// Stereotype annotations — all are @Component specialisations
@Component      // generic Spring-managed bean
@Service        // business logic layer (semantic — no special behaviour)
@Repository     // data access layer (adds exception translation: SQL → Spring DataAccessException)
@Controller     // web layer (returns views)
@RestController // web layer + @ResponseBody on every method (returns JSON)

// @Configuration + @Bean — for third-party classes you can't annotate
@Configuration
public class AppConfig {
    @Bean                                       // method return value becomes a bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(5))
            .build();
    }
    // ⚠️ @Bean methods in @Configuration are proxied — calling them again returns same singleton
    // In @Component class, @Bean methods are NOT proxied — each call creates new instance
}
```

### 6. @Qualifier, @Primary, and Profiles

```java
// Multiple beans of same type — Spring doesn't know which to inject
public interface PaymentGateway { }

@Component("stripe")
public class StripeGateway implements PaymentGateway { }

@Component("paypal")
@Primary                              // used by default when no qualifier specified
public class PayPalGateway implements PaymentGateway { }

@Service
public class PaymentService {
    // Uses @Primary (PayPal) — no qualifier needed
    public PaymentService(PaymentGateway gateway) { }
    
    // OR: explicitly select Stripe
    public PaymentService(@Qualifier("stripe") PaymentGateway gateway) { }
}

// Profiles — environment-specific beans
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() { return new H2DataSource(); }  // embedded for dev
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() { return new PgDataSource(); }  // PostgreSQL for prod
}
// Activate: spring.profiles.active=dev (in application.yml, env var, or CLI)
```

### 7. @ConfigurationProperties — Type-Safe Config

```java
// application.yml
// app:
//   trade:
//     max-quantity: 10000
//     timeout: 30s
//     allowed-symbols:
//       - AAPL
//       - GOOG

@ConfigurationProperties(prefix = "app.trade")
@Validated                                          // enables Bean Validation
public class TradeProperties {
    @Min(1) private int maxQuantity;
    private Duration timeout;                       // auto-converts "30s" → Duration
    @NotEmpty private List<String> allowedSymbols;
    // getters/setters
}

// Enable in config class: @EnableConfigurationProperties(TradeProperties.class)
// Or use: @ConfigurationPropertiesScan

// vs @Value — for single properties (no validation, no grouping)
@Value("${app.trade.max-quantity}")
private int maxQuantity;
```

### 8. Circular Dependencies

```java
// A depends on B, B depends on A
@Service
public class A {
    public A(B b) { }   // needs B
}
@Service
public class B {
    public B(A a) { }   // needs A — CIRCULAR!
}

// With CONSTRUCTOR injection → fails at startup: BeanCurrentlyInCreationException
// This is GOOD — circular dependency is a design smell

// With SETTER/FIELD injection → Spring resolves it using three-level cache:
// 1. Create A (partially — no deps yet)
// 2. Expose A's early reference in cache
// 3. Create B, inject early A reference
// 4. Complete A by injecting B

// ✅ Proper fix: Break the cycle
// Option 1: Extract shared logic into a third service
// Option 2: Use @Lazy on one dependency (defers proxy creation)
// Option 3: Use events instead of direct dependency
@Service
public class A {
    public A(@Lazy B b) { }   // injects a proxy — real B created on first use
}
```

---

## Rapid-Fire Q&A

### Q1: What's IoC (Inversion of Control)?
**A:** The framework controls object creation and wiring — not your code. You declare dependencies, the container provides them. Spring's IoC container manages the full bean lifecycle: creation → dependency injection → initialisation → destruction.

### Q2: What's Dependency Injection? Three types?
**A:** Container provides dependencies to a class. Three types: (1) **Constructor injection** (preferred — makes dependencies explicit, supports immutability, required deps). (2) **Setter injection** (optional deps). (3) **Field injection** (`@Autowired` on field — discouraged, can't use in tests without reflection).

### Q3: Why is constructor injection preferred?
**A:** (1) Dependencies are explicit — no hidden coupling. (2) Fields can be `final` — immutable. (3) Easier to test — pass mocks via constructor. (4) Circular dependencies fail fast at startup instead of at runtime. Since Spring 4.3, single constructor doesn't even need `@Autowired`.

### Q4: What are bean scopes?
**A:** `singleton` (default — one instance per container), `prototype` (new instance per injection), `request` (per HTTP request), `session` (per HTTP session), `application` (per ServletContext).

### Q5: How do you inject a Prototype bean into a Singleton?
**A:** Direct injection gives you one instance forever. Solutions: (1) `@Lookup` method — Spring overrides it via CGLIB to return fresh prototype. (2) `ObjectFactory<T>` or `Provider<T>` injection — call `.getObject()` each time. (3) `ApplicationContext.getBean()` — service locator (less clean).

### Q6: What's `@Qualifier`?
**A:** Disambiguates when multiple beans of the same type exist. `@Autowired @Qualifier("stripe")` selects the bean named "stripe". Alternative: `@Primary` marks a default bean.

### Q7: Bean lifecycle — what hooks exist?
**A:** Construction → `@PostConstruct` (or `InitializingBean.afterPropertiesSet()`) → ready → `@PreDestroy` (or `DisposableBean.destroy()`). Also: `BeanPostProcessor` for cross-cutting init logic. `@PostConstruct` runs after all dependencies are injected.

### Q8: What are Spring profiles?
**A:** Environment-specific configuration. `@Profile("dev")` activates bean only in dev. `application-dev.yml` loaded when `spring.profiles.active=dev`. Common: dev, staging, prod profiles with different datasource, logging, feature flags.

### Q9: `@ConfigurationProperties` vs `@Value`?
**A:** `@Value("${app.timeout}")` — single property, no type safety. `@ConfigurationProperties(prefix="app")` — binds a whole group to a POJO. Preferred for structured config — type-safe, IDE support, validation with `@Validated`.

### Q10: How does Spring resolve circular dependencies?
**A:** With setter/field injection: Spring creates a partially initialised bean, injects it, then completes initialisation (three-level cache). With constructor injection: fails at startup — which is actually good (design smell). Fix: break the cycle with `@Lazy` or restructure.

### Q11: What's `@Component` vs `@Service` vs `@Repository` vs `@Controller`?
**A:** All are `@Component` stereotypes — Spring auto-detects them via component scan. Semantic difference: `@Service` → business logic, `@Repository` → data access (adds exception translation), `@Controller` → web layer. Use the right one for clarity.

### Q12: What's `@Configuration` and `@Bean`?
**A:** `@Configuration` class contains `@Bean` factory methods. Each `@Bean` method returns an object registered in the container. Used for third-party classes you can't annotate with `@Component`. Methods are proxied — calling a `@Bean` method again returns the same singleton.

---

## Can you answer these cold?

- [ ] Three DI types — constructor (preferred), setter, field
- [ ] Bean scopes — singleton, prototype, request, session
- [ ] Prototype into singleton problem — three solutions
- [ ] Bean lifecycle — `@PostConstruct` / `@PreDestroy`
- [ ] Circular dependency — how Spring resolves it, why constructor injection fails
- [ ] `@ConfigurationProperties` — why preferred over `@Value`

[← Back to Index](./00_INDEX.md)
