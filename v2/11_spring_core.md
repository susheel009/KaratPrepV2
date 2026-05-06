# 11 — Spring Core & Dependency Injection

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/11_spring_core_doubts.md) · [← Prev: 10 Testing](./10_testing.md) · [Next: 12 Spring Boot →](./12_spring_boot.md)
>
> **Priority:** 🟡 High · **Related topics:** [12 Spring Boot](./12_spring_boot.md) · [14 Spring AOP & Data](./14_spring_aop_data.md) · [09 SOLID (DIP)](./09_solid.md) · [08 Design Patterns (Proxy, Factory)](./08_design_patterns.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — wiring without writing wiring code
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — IoC container builds the graph
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — minimal Spring app
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 IoC + DI](#41-ioc--di--what-they-actually-mean)
   - [4.2 Three injection types](#42-three-injection-types)
   - [4.3 Beans + stereotypes](#43-beans-and-stereotype-annotations)
   - [4.4 @Configuration + @Bean](#44-configuration--bean--for-third-party-classes)
   - [4.5 Bean scopes](#45-bean-scopes)
   - [4.6 Bean lifecycle](#46-bean-lifecycle)
   - [4.7 @Qualifier + @Primary + Profiles](#47-qualifier-primary-and-profiles)
   - [4.8 @ConfigurationProperties](#48-configurationproperties--type-safe-config)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 Why constructor injection wins](#51-why-constructor-injection-wins)
   - [5.2 Prototype-in-singleton problem](#52-the-prototype-in-singleton-problem)
   - [5.3 Circular dependencies](#53-circular-dependencies)
   - [5.4 BeanPostProcessor](#54-beanpostprocessor--how-spring-adds-cross-cutting)
   - [5.5 @Configuration vs @Component @Bean](#55-configuration-vs-component-bean)
   - [5.6 ApplicationContext hierarchy](#56-applicationcontext-hierarchy-and-startup-phases)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

You're hired to write the office for a small bank. The first task: an `OrderService` that needs an `OrderRepository`, an `EmailService`, an `AuditLogger`, and a `PaymentGateway`.

Without help, your code starts like this: at the top of `OrderService`, you `new MySQLOrderRepository(...)`, you `new SmtpEmailService(...)`, you `new FileAuditLogger(...)`, you `new StripePaymentGateway(...)`. Each of *those* classes has its own dependencies — a connection pool, an SMTP host, a file path, an API key. You either hard-code them or thread them in. Now multiply across 200 classes in a real app. Every one of them is *both* doing its own job *and* knowing how to construct the world around it. Tests are agony — to construct one `OrderService`, you have to construct half the application.

Spring's idea: there's a **single component** at the centre — an **IoC container** — whose only job is *building the graph of objects*. You declare what each class needs (dependencies) and what kind of class each is (a service, a repo, a controller). The container reads these declarations at startup, builds every object, and wires them together. Your classes never call `new MySQLOrderRepository(...)` again — they just declare a constructor parameter `OrderRepository repo`, and the container hands them one.

This single decision — **the container builds the graph** — is the core of Spring. Everything else (Boot, MVC, Data, Security, AOP) is a thick layer of capabilities that *use* the container to deliver their value.

> Once you see Spring as "a container that builds the graph + a pile of capabilities that depend on having a container", every Spring annotation becomes a hint to the container ("this class is a service", "this method produces a bean", "inject this here") — not arbitrary ceremony.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

`OrderService` needs an `OrderRepository`. `OrderRepository` is implemented by `JpaOrderRepository`, which itself needs a `DataSource`.

What the container does at startup:

| Step | What |
|---|---|
| 1 | Scan classpath for annotated classes. Finds `OrderService` (`@Service`), `JpaOrderRepository` (`@Repository`), `DataSource` bean (defined in a `@Configuration` class). |
| 2 | Build a graph: `OrderService` → `OrderRepository` → `DataSource`. |
| 3 | Resolve in topological order. First instantiate the `DataSource` (no deps). Then `JpaOrderRepository(DataSource)`. Then `OrderService(OrderRepository)`. |
| 4 | After all are constructed, run any `@PostConstruct` methods. |
| 5 | The application is "ready". Your `main` returns control or starts the web server. |

What this gives you:

1. Your high-level code (`OrderService`) names only the **interface** it needs (`OrderRepository`). It never sees `JpaOrderRepository` or `DataSource`. (This is **DIP** — see [09 §4.5](./09_solid.md#45-dip--dependency-inversion).)
2. Swapping implementations is a one-line change in a configuration — no edits to `OrderService`. In tests: pass a mock instead of `JpaOrderRepository`.
3. The container handles cross-cutting concerns — transactions, security, caching — by *wrapping* your beans in proxies (see [08 §4.6](./08_design_patterns.md#46-proxy--control-access-or-add-cross-cutting-behaviour) and [14 Spring AOP & Data](./14_spring_aop_data.md)).

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
// === Interface — what OrderService depends on ===
public interface OrderRepository {
    void save(Order o);
}

// === Implementation — Spring discovers it via @Repository ===
@Repository                                                    // Spring stereotype = "data access bean"
public class JpaOrderRepository implements OrderRepository {
    private final EntityManager em;
    public JpaOrderRepository(EntityManager em) {              // constructor injection
        this.em = em;
    }
    public void save(Order o) { em.persist(o); }
}

// === The service — depends on OrderRepository, NOT a concrete impl ===
@Service                                                       // Spring stereotype = "business logic"
public class OrderService {
    private final OrderRepository repo;                         // depends on the interface
    public OrderService(OrderRepository repo) {                 // constructor injection
        this.repo = repo;
    }
    public void place(Order o) { repo.save(o); }
}

// === Spring Boot bootstraps the container ===
@SpringBootApplication                                         // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class App {
    public static void main(String[] args) {
        // 1. ApplicationContext starts.
        // 2. Component scan finds @Service, @Repository, @Component, @Controller.
        // 3. Spring builds the graph: DataSource → JpaOrderRepository → OrderService.
        // 4. main returns; web server takes over (or batch job runs).
        SpringApplication.run(App.class, args);
    }
}
```

What just happened: `OrderService` says "I need an `OrderRepository`". Spring scans, finds `JpaOrderRepository implements OrderRepository`, builds it (with its own deps), and passes it into `OrderService`'s constructor. **No `new`s** in your high-level code.

---

## 4. Build Up — Practical Patterns

### 4.1 IoC + DI — what they actually mean

**Inversion of Control (IoC).** "Don't call us, we'll call you." Your classes don't construct their dependencies; the container does. Control of object lifecycle is *inverted* — moved from your code to the framework.

**Dependency Injection (DI).** A specific technique for IoC: dependencies are *injected* into the class (via constructor, setter, or field). The class declares what it needs and receives it from outside.

```java
// WITHOUT IoC — your code creates everything (tight coupling)
public class OrderService {
    private final OrderRepository repo = new MySQLOrderRepository();    // hard-wired
    private final EmailService email = new SmtpEmailService();          // hard-wired
}

// WITH IoC — container provides the dependencies
@Service
public class OrderService {
    private final OrderRepository repo;
    private final EmailService email;
    public OrderService(OrderRepository repo, EmailService email) {     // ask, don't build
        this.repo = repo; this.email = email;
    }
}
```

> 💡 Spring's IoC is **literally** SOLID's DIP made concrete. The high-level `OrderService` depends on the abstraction `OrderRepository`; both depend on neither's implementation. Spring is the wiring tool.

### 4.2 Three injection types

```java
// 1. CONSTRUCTOR INJECTION — preferred
@Service
public class TradeService {
    private final TradeRepository repo;       // final → immutable
    private final AuditService    audit;
    public TradeService(TradeRepository repo, AuditService audit) {     // since Spring 4.3, no @Autowired needed for single ctor
        this.repo = repo; this.audit = audit;
    }
}
// ✅ Dependencies explicit, fields can be final, easy to test (no Spring needed)

// 2. SETTER INJECTION — for OPTIONAL dependencies
@Service
public class ReportService {
    private CacheService cache;                 // optional
    @Autowired(required = false)                // doesn't fail if no CacheService bean
    public void setCache(CacheService cache) { this.cache = cache; }
}

// 3. FIELD INJECTION — discouraged
@Service
public class PaymentService {
    @Autowired private PaymentGateway gateway;  // hidden dependency, can't be final
}
// ❌ Hides deps, prevents final, hard to test without reflection or full Spring context
```

### 4.3 Beans and stereotype annotations

A **bean** is just an object Spring manages.

```java
@Component       // generic — any Spring-managed class
@Service         // semantic: business logic (no special behaviour over @Component)
@Repository      // semantic: data access (adds JDBC exception translation)
@Controller      // semantic: web request handler (returns view names)
@RestController  // = @Controller + @ResponseBody on every method (returns JSON)
```

All four "semantic" stereotypes are `@Component` underneath. Use the most specific one for clarity. `@Repository` *does* add behaviour: it converts SQL/JPA exceptions into Spring's `DataAccessException` hierarchy.

### 4.4 `@Configuration` + `@Bean` — for third-party classes

Sometimes a class isn't yours to annotate (a JDK class, a third-party library). Define a bean by writing a factory method in a `@Configuration` class:

```java
@Configuration
public class AppConfig {
    @Bean
    public RestClient restClient() {
        return RestClient.builder()
            .baseUrl("https://api.example.com")
            .build();
    }
    @Bean
    public Clock systemClock() { return Clock.systemUTC(); }
}
```

Each `@Bean` method is invoked once at startup; the return value is a singleton in the container.

### 4.5 Bean scopes

```java
@Component @Scope("singleton")    // DEFAULT — one instance per ApplicationContext
@Component @Scope("prototype")    // new instance every time it's injected/looked up
@Component @Scope("request")      // one per HTTP request (web only)
@Component @Scope("session")      // one per HTTP session (web only)
@Component @Scope("application")  // one per ServletContext
```

The default — **singleton scope** — is what 95% of Spring beans should be. Stateless services, repositories, and controllers are all singletons.

### 4.6 Bean lifecycle

```
new (ctor)  →  dependencies injected  →  @PostConstruct  →  ready  →  @PreDestroy (on shutdown)
```

```java
@Service
public class CacheService {
    @PostConstruct void init()     { warmCache();  log.info("warm"); }   // after deps injected
    @PreDestroy   void cleanup()   { flushCache(); log.info("flushed"); } // on shutdown
}
```

Modern alternative — implement `InitializingBean.afterPropertiesSet()` and `DisposableBean.destroy()`. The annotation form is preferred (less coupling to Spring types).

### 4.7 `@Qualifier`, `@Primary`, and Profiles

When two beans implement the same interface, Spring needs help choosing:

```java
@Component("stripe") public class StripeGateway implements PaymentGateway {}
@Component("paypal") @Primary public class PayPalGateway implements PaymentGateway {}

// Default — uses @Primary (PayPal)
public PaymentService(PaymentGateway gateway) { /* ... */ }

// Explicit — by name
public PaymentService(@Qualifier("stripe") PaymentGateway gateway) { /* ... */ }
```

**Profiles** swap entire bean sets per environment:

```java
@Configuration @Profile("dev")
public class DevConfig { @Bean public DataSource ds() { return new H2DataSource(); } }

@Configuration @Profile("prod")
public class ProdConfig { @Bean public DataSource ds() { return new PostgresDataSource(); } }

// Activate with: spring.profiles.active=dev (in application.yml, env var, or CLI)
```

### 4.8 `@ConfigurationProperties` — type-safe config

```yaml
# application.yml
app:
  trade:
    max-quantity: 10000
    timeout: 30s
    allowed-symbols: [AAPL, GOOG, MSFT]
```

```java
@ConfigurationProperties(prefix = "app.trade")
@Validated                                        // enables Bean Validation
public record TradeProperties(
    @Min(1) int maxQuantity,
    Duration timeout,                             // Spring auto-converts "30s" → Duration
    @NotEmpty List<String> allowedSymbols
) {}

// Enable: @ConfigurationPropertiesScan or @EnableConfigurationProperties(TradeProperties.class)
```

Compare with `@Value("${app.trade.max-quantity}")` — single property, no grouping, no validation. Prefer `@ConfigurationProperties` for any non-trivial config.

---

## 5. Going Deep — Interview-Level Material

### 5.1 Why constructor injection wins

The Spring team officially recommends constructor injection for four reasons:

1. **Explicit dependencies.** They appear in the constructor signature — visible at a glance, can't be forgotten.
2. **Immutability.** Fields can be `final`. Once constructed, the object's collaborator graph cannot change.
3. **Testability.** No Spring needed in the test — `new OrderService(mockRepo, mockAudit)`. Field injection requires reflection or a full context.
4. **Fails fast on circular deps.** A circular dependency through constructors throws at startup — *before* the application accepts traffic. Field/setter injection silently resolves circularity using the three-level cache, leaving you with a partially-initialised bean that surprises someone at runtime.

Bonus: since Spring 4.3, a class with **a single constructor** doesn't need `@Autowired` — the framework picks it automatically. Result: pure-Java code with no Spring imports in your domain classes.

### 5.2 The prototype-in-singleton problem

```java
@Service                                          // singleton (default)
public class OrderService {
    @Autowired private RequestProcessor processor;  // prototype — but injected ONCE
}
```

`OrderService` is a singleton. The `processor` field is set once at startup and never refreshed — even though `@Scope("prototype")` says "new instance per injection". Three workarounds:

```java
// 1. @Lookup — Spring overrides the abstract method via CGLIB
@Service
public abstract class OrderService {
    @Lookup
    abstract RequestProcessor newProcessor();      // returns a fresh prototype on each call
}

// 2. Inject ObjectFactory<T> / Provider<T> — call .getObject() each time
@Service
public class OrderService {
    private final ObjectFactory<RequestProcessor> processorFactory;
    public OrderService(ObjectFactory<RequestProcessor> f) { this.processorFactory = f; }
    public void run() { var p = processorFactory.getObject(); /* fresh */ }
}

// 3. ApplicationContext.getBean(...) — service-locator anti-pattern; avoid unless trapped
```

### 5.3 Circular dependencies

```java
@Service public class A { public A(B b) {} }      // needs B
@Service public class B { public B(A a) {} }      // needs A — CIRCLE
```

With **constructor injection**, this fails at startup with `BeanCurrentlyInCreationException`. **This is good** — it's a design smell forcing you to fix the architecture.

With **setter / field injection**, Spring solves it via a three-level cache:
1. Construct A (no deps yet).
2. Expose A's *early* reference in the cache.
3. Construct B, inject the early A reference.
4. Complete A by injecting B.

The result is technically working but fragile — A and B are now mutually entangled, and code that runs in A's constructor can't see B yet. The proper fix is to **break the cycle**:

```java
// Option A: extract the shared logic into a third bean.
// Option B: invert one direction with an event (publisher → listener instead of direct call).
// Option C: @Lazy on one dependency — Spring injects a proxy; real bean created on first use
@Service
public class A {
    public A(@Lazy B b) {}             // proxy now, real B later
}
```

### 5.4 BeanPostProcessor — how Spring adds cross-cutting

A `BeanPostProcessor` is a hook the container runs *for every bean*, twice (before and after init callbacks). Spring's own annotations are implemented via post-processors:

```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName);   // before @PostConstruct
    Object postProcessAfterInitialization(Object bean, String beanName);    // after @PostConstruct
}
```

Spring's internal post-processors handle:

- `@Autowired` field/setter resolution
- `@PostConstruct` / `@PreDestroy` (`CommonAnnotationBeanPostProcessor`)
- `@Transactional` — wraps eligible beans in a CGLIB/JDK proxy
- `@Async`, `@Cacheable`, `@PreAuthorize` — same trick

Knowing this explains why **self-invocation skips the proxy** ([08 §4.6](./08_design_patterns.md#46-proxy--control-access-or-add-cross-cutting-behaviour)) — `this.method()` calls the unwrapped instance, not the proxy returned from the post-processor.

You rarely write your own `BeanPostProcessor`, but seeing one in a stack trace or startup log shouldn't surprise you.

### 5.5 `@Configuration` vs `@Component @Bean`

A subtle but interview-relevant distinction. Both can hold `@Bean` methods:

```java
@Configuration                       // CGLIB-proxied at class level
public class AppConfig {
    @Bean public Foo foo() { return new Foo(); }
    @Bean public Bar bar() { return new Bar(foo()); }    // foo() returns the SAME singleton
}

@Component                           // NOT class-level proxied
public class AppComponent {
    @Bean public Foo foo() { return new Foo(); }
    @Bean public Bar bar() { return new Bar(foo()); }    // foo() returns a NEW Foo each call
}
```

Inside a `@Configuration` class, calls to `@Bean` methods are intercepted by a CGLIB proxy that returns the singleton from the container. Inside a `@Component` (or "lite" mode), they're plain method calls — every invocation creates a new instance. **Always use `@Configuration` for classes that hold `@Bean` methods.**

### 5.6 ApplicationContext hierarchy and startup phases

`ApplicationContext` is the container interface. Its lifecycle has rough phases:

1. **Constructor** — `SpringApplication.run(...)`.
2. **BeanDefinitionLoading** — scan and parse `@Component`/`@Bean`/XML/Java config into `BeanDefinition`s.
3. **BeanFactoryPostProcessing** — modify definitions (e.g. property placeholder resolution).
4. **Bean instantiation** — build singletons in dependency order.
5. **Bean initialization** — `@Autowired`, `@PostConstruct`.
6. **Refresh complete** — context fires `ContextRefreshedEvent`. Web servers start listening here.
7. **Runtime** — beans serve traffic.
8. **Close** — `@PreDestroy`, then `ContextClosedEvent`.

In a Spring MVC application (web), there's a **two-context** pattern: a *root* context (services, repositories) shared across all servlets, and a *web* context per `DispatcherServlet` (controllers, view resolvers) with the root as parent. Spring Boot mostly hides this — single context unless you're integrating an old XML-driven web app.

---

## 6. Memory Aids

### Decision tree: "how do I provide this dependency to my class?"

```
Is the dependency required for the class to function?
├── Yes → CONSTRUCTOR injection.
└── No (truly optional) → SETTER injection (@Autowired(required = false)).

Is the dependency from a third-party class you can't annotate?
└── Yes → @Bean method in a @Configuration class.

Are there multiple beans of the same type?
├── Has a clear "default" → @Primary on the default.
├── Need to choose at injection point → @Qualifier("name").
└── Differs per environment → @Profile("dev"/"prod") on each.

Is this configuration data (URLs, sizes, timeouts)?
└── @ConfigurationProperties(prefix = "app.x") on a record.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "What's IoC?" | Container builds the graph | "Your code declares deps; the container constructs and wires them. DI is the technique for IoC." |
| "Why constructor injection?" | Explicit + final + testable + circular-fail-fast | "Four wins. Field injection has none." |
| "Default bean scope?" | Singleton | "One per ApplicationContext. Used for ~95% of beans." |
| "Prototype into singleton — what breaks?" | Injected once, never refreshed | "Use `ObjectFactory<T>` or `@Lookup`." |
| "Circular dependency with constructors?" | Fails at startup | "Good — design smell. Break the cycle (extract / event / @Lazy)." |
| "What does `@Repository` add over `@Component`?" | Exception translation | "Converts JPA / SQL exceptions into Spring's `DataAccessException` hierarchy." |
| "`@Configuration` vs `@Component` for `@Bean` methods?" | Proxied vs plain | "@Configuration intercepts calls to return the singleton. @Component creates a new instance per call." |
| "@ConfigurationProperties vs @Value?" | Group + type-safe vs single | "@ConfigurationProperties for any non-trivial config." |

### Three anchor pictures

1. **Container builds the graph.** You declare; it constructs.
2. **Constructor injection.** Explicit, final, testable, fails fast.
3. **Singleton by default.** Stateless services and repositories live for the life of the context.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: What's IoC?
**A:** Inversion of Control — the framework controls object creation and wiring instead of your code. You declare dependencies; the container provides them. Spring's `ApplicationContext` is the IoC container.

### Q2: What's DI? Three types?
**A:** Dependency Injection — the container provides dependencies to a class. Constructor (preferred — explicit, final, testable), setter (optional dependencies), field (`@Autowired` on field — discouraged, hides deps, prevents final).

### Q3: Why is constructor injection preferred?
**A:** (1) Dependencies are explicit in the signature. (2) Fields can be `final` — immutable. (3) Easy to test — pass mocks via constructor. (4) Circular dependencies fail fast at startup. Since Spring 4.3, a single constructor doesn't need `@Autowired`.

### Q4: Bean scopes?
**A:** `singleton` (default — one per ApplicationContext), `prototype` (new instance per injection), `request` (per HTTP request, web only), `session` (per HTTP session), `application` (per ServletContext).

### Q5: How do you inject a Prototype into a Singleton correctly?
**A:** Direct `@Autowired` injects once and never refreshes. Solutions: `@Lookup` method (Spring overrides via CGLIB to return fresh prototype), `ObjectFactory<T>` / `Provider<T>` (`.getObject()` returns fresh each call), or `ApplicationContext.getBean(...)` (service-locator — last resort).

### Q6: What's `@Qualifier`?
**A:** Disambiguates when multiple beans of the same type exist. `@Autowired @Qualifier("stripe") PaymentGateway` selects the bean named "stripe". Alternative: mark one `@Primary` to make it the default.

### Q7: Bean lifecycle hooks?
**A:** Construction → dependencies injected → `@PostConstruct` (or `InitializingBean.afterPropertiesSet()`) → ready → `@PreDestroy` (or `DisposableBean.destroy()`) on shutdown. `BeanPostProcessor` runs around init for cross-cutting concerns.

### Q8: What are Spring profiles?
**A:** Environment-specific bean activation. `@Profile("dev")` activates a bean only when `spring.profiles.active=dev`. Common: dev, staging, prod with different DataSource, security, feature flags. Activated via env var, CLI arg, or `application.yml`.

### Q9: `@ConfigurationProperties` vs `@Value`?
**A:** `@Value("${prop}")` — single property, no type-safety, no validation. `@ConfigurationProperties(prefix = "app")` — binds a whole group to a record/POJO, supports `Duration`/`DataSize` conversion, supports `@Validated`. Prefer for non-trivial config.

### Q10: How does Spring resolve circular dependencies?
**A:** With setter/field injection, the three-level cache: construct A partially → expose A's early reference → construct B with that early ref → complete A. With constructor injection, fails at startup — which is good (design smell). Fix: break the cycle (extract shared logic, event, `@Lazy`).

### Q11: `@Component` vs `@Service` vs `@Repository` vs `@Controller`?
**A:** All are `@Component` stereotypes — Spring auto-detects via component scan. Semantic differences: `@Service` = business logic, `@Repository` = data access (adds exception translation), `@Controller` = web request handler (returns view names), `@RestController` = `@Controller` + `@ResponseBody`.

### Q12: `@Configuration` and `@Bean`?
**A:** `@Configuration` class contains `@Bean` factory methods. Each `@Bean` method's return value is registered as a bean. Used for third-party classes you can't annotate. Crucial: `@Configuration` proxies the class via CGLIB, so calling one `@Bean` method from another returns the same singleton (vs `@Component` where it creates a new instance).

### Q13: What does `@SpringBootApplication` include?
**A:** `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Marks the class as a config source, enables Boot's conditional auto-configuration, and scans the current package + subpackages for components.

### Q14: What's `BeanPostProcessor`?
**A:** A hook the container runs around every bean's init phase. Used internally by Spring for `@Autowired` resolution, `@Transactional` proxy creation, `@Async`, `@Cacheable`. Rarely written by application code, but explains how cross-cutting concerns are added.

### Q15: How does Spring decide which constructor to use?
**A:** If there's exactly one constructor, Spring uses it (no `@Autowired` needed since 4.3). If multiple, mark one `@Autowired`. With Lombok's `@RequiredArgsConstructor`, the generated single constructor is auto-detected.

### Q16: When would you use a custom `BeanPostProcessor`?
**A:** Implementing custom annotations that add cross-cutting behaviour (e.g. injecting a custom audit logger into every bean), bridging a third-party library's lifecycle into Spring's. Rare in application code.

---

### Key Code Patterns

**Standard constructor injection**
```java
@Service
public class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) { this.repo = repo; }    // no @Autowired needed
}
```

**`@Configuration` for third-party class**
```java
@Configuration
public class HttpConfig {
    @Bean public RestClient restClient() {
        return RestClient.builder().baseUrl(props.url()).build();
    }
}
```

**`@ConfigurationProperties` record**
```java
@ConfigurationProperties("app.trade")
@Validated
public record TradeProps(@Min(1) int maxQuantity, Duration timeout) {}
```

**Profile-specific bean**
```java
@Configuration @Profile("prod")
public class ProdDb {
    @Bean DataSource dataSource() { return new HikariDataSource(prodConfig()); }
}
```

**Prototype factory**
```java
public OrderService(ObjectFactory<RequestProcessor> processorFactory) { /* getObject() per use */ }
```

---

## 8. Self-Test

**Easy**
- [ ] What's the difference between IoC and DI?
- [ ] What's the default bean scope?
- [ ] What's `@Component` vs `@Service`?
- [ ] What does `@SpringBootApplication` combine?

**Medium**
- [ ] Trace the §2 walkthrough — what does the container do at startup?
- [ ] Why is constructor injection preferred over field injection?
- [ ] When would you use `@Qualifier` vs `@Primary` vs `@Profile`?
- [ ] What's the prototype-in-singleton problem and three ways to solve it?

**Hard**
- [ ] Why does a circular dependency through constructors fail at startup?
- [ ] What's the difference between `@Configuration @Bean` and `@Component @Bean`?
- [ ] Walk through what `BeanPostProcessor` does. Name two of Spring's own post-processors.
- [ ] How would you bind a YAML config block to a record with validation?
- [ ] In a multi-module app, how do you share a base config across profiles?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **IoC** | Inversion of Control — framework runs your code, not the other way round. |
| **DI** | Dependency Injection — dependencies are passed in, not created internally. |
| **ApplicationContext** | The Spring IoC container. |
| **Bean** | An object managed by the container. |
| **Stereotype** | An annotation marking a bean's role (`@Service`, `@Repository`, `@Controller`). |
| **`@Configuration`** | Class containing `@Bean` factory methods; class-level CGLIB-proxied. |
| **`@Bean`** | Method whose return value is registered as a bean. |
| **Scope** | A bean's lifetime (singleton, prototype, request, session, application). |
| **`@PostConstruct`** | Method run after dependencies are injected. |
| **`@PreDestroy`** | Method run on shutdown. |
| **Component scan** | Classpath scan for stereotype-annotated classes. |
| **Auto-wiring** | The container's process of matching dependencies to beans. |
| **`@Qualifier`** | Disambiguator when multiple beans of the same type exist. |
| **`@Primary`** | Default bean when multiple of the same type exist. |
| **Profile** | Named set of beans/config activated per environment. |
| **`@ConfigurationProperties`** | Type-safe binding of a YAML/properties prefix to a POJO/record. |
| **`BeanPostProcessor`** | Hook running around every bean's init — Spring's mechanism for cross-cutting. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/11_spring_core_doubts.md) · [← Prev: 10 Testing](./10_testing.md) · [Next: 12 Spring Boot →](./12_spring_boot.md)

[↑ Back to top](#11--spring-core--dependency-injection)
