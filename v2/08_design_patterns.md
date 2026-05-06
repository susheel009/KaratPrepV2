# 08 — Design Patterns

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/08_design_patterns_doubts.md) · [← Prev: 07 JVM](./07_jvm.md) · [Next: 09 SOLID →](./09_solid.md)
>
> **Priority:** 🟢 Medium · **Related topics:** [09 SOLID](./09_solid.md) · [11 Spring Core](./11_spring_core.md) · [03 OOP](./03_oop_language.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — why we need named recipes for code shapes
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — Strategy via `Comparator`
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — Strategy in 15 lines
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Singleton](#41-singleton--exactly-one-instance)
   - [4.2 Factory & Abstract Factory](#42-factory-method-and-abstract-factory)
   - [4.3 Builder](#43-builder--complex-construction-step-by-step)
   - [4.4 Strategy](#44-strategy--swap-algorithms-at-runtime)
   - [4.5 Observer](#45-observer--one-to-many-event-notification)
   - [4.6 Proxy](#46-proxy--control-access-or-add-cross-cutting-behaviour)
   - [4.7 Decorator](#47-decorator--add-features-by-wrapping)
   - [4.8 Adapter](#48-adapter--bridge-incompatible-interfaces)
   - [4.9 Template Method](#49-template-method--algorithm-skeleton-with-pluggable-steps)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 When patterns are over-engineering](#51-when-patterns-are-over-engineering)
   - [5.2 How Spring uses them](#52-how-spring-uses-the-patterns-internally)
   - [5.3 Modern Java replacements](#53-modern-java-replacements-for-old-pattern-boilerplate)
   - [5.4 Patterns that combine](#54-patterns-that-combine)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Imagine you're a chef opening a new restaurant. You hire five other chefs. None of them have ever cooked your menu before. On day one, every chef invents their own way to:

- chop an onion
- plate a salad
- communicate with the dishwasher
- handle a cancelled order

By dinner service, the kitchen is chaos. Five techniques for the same outcome, none documented, each chef thinks the others are doing it wrong.

The next morning you write a small **recipe book** — not the food recipes, but the *kitchen-process* recipes. "When two chefs need the same fryer, here's how they coordinate." "When you finish a dish, here's how you signal the runner." Each recipe has a name, a stated problem it solves, and the canonical solution. Now when one chef says to another *"use the call-back recipe"*, everybody knows exactly what to do.

That's a **design pattern**. Not a snippet of code, not a library — a **named, reusable solution to a recurring problem**. The Gang of Four book (1994) catalogued the dozen-or-so most common ones in OOP code. Decades later, you'll still hear engineers say "we used a Strategy here", "Spring builds beans via Factory", "wrap that with a Decorator" — and the listener immediately understands.

The patterns you'll be asked about in interviews divide into three rough buckets:

- **Creational** — recipes for *making* objects: Singleton (only one), Factory (decide which subtype), Builder (assemble step by step).
- **Structural** — recipes for *combining* objects: Proxy (stand-in), Decorator (wrap to add features), Adapter (bridge two incompatible APIs).
- **Behavioural** — recipes for *interaction*: Strategy (swap algorithm), Observer (broadcast events), Template Method (parent defines the steps, child fills the blanks).

> Once you can name the problem each pattern solves, you can read any modern Java codebase like a recipe book — Spring, JDK I/O, Servlets, and most Spring Boot starters are built almost entirely from these.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

A list of `Employee` objects. Sort them sometimes by name, sometimes by salary, sometimes by start date. Without a pattern, you'd write three different sort methods.

With **Strategy**, you write **one** sort and pass in the comparison rule:

```
sort(employees, "first by name")             // strategy = a function(a, b) -> compare names
sort(employees, "first by salary descending")// strategy = a function(a, b) -> compare salaries reversed
sort(employees, "first by hire date")        // strategy = a function(a, b) -> compare dates
```

The function-that-compares **is** the strategy. The sort algorithm doesn't change. Different strategies in, different orderings out.

Two takeaways the walkthrough makes obvious:

1. The **strategy is a value** — passed in as an argument, swappable at runtime.
2. The **caller picks**, not the algorithm. The algorithm is open for new orderings without modification.

In Java, `Comparator` *is* the Strategy interface, and lambdas make passing one in trivial. Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
import java.util.*;
import java.util.function.*;

public class StrategyDemo {
    record Employee(String name, double salary, String dept) {}

    public static void main(String[] args) {
        var employees = new ArrayList<>(List.of(
            new Employee("Bob",   60_000, "ENG"),
            new Employee("Alice", 90_000, "ENG"),
            new Employee("Carol", 75_000, "PR")
        ));

        // STRATEGY = a Comparator. Pass a different one for a different ordering.
        Comparator<Employee> byName     = Comparator.comparing(Employee::name);
        Comparator<Employee> bySalary   = Comparator.comparingDouble(Employee::salary).reversed();
        Comparator<Employee> byDeptThenName =
            Comparator.comparing(Employee::dept).thenComparing(Employee::name);

        // SAME sort algorithm, three different strategies → three different results.
        employees.sort(byName);            System.out.println(employees);
        employees.sort(bySalary);          System.out.println(employees);
        employees.sort(byDeptThenName);    System.out.println(employees);
    }
}
```

What just happened: `List.sort` doesn't know whether you're ordering by name, salary, or hire date — it just calls the strategy you passed. Each `Comparator` is a tiny value-typed strategy. To add a new ordering, write a new comparator. **No modification to the sorting code.**

---

## 4. Build Up — Practical Patterns

### 4.1 Singleton — exactly one instance

**Intent:** ensure a class has exactly one instance with a global access point.

```java
// BEST — enum singleton (thread-safe, serialisation-safe, no boilerplate)
public enum AppConfig {
    INSTANCE;
    private final Properties props = loadProps();
    public String get(String key) { return props.getProperty(key); }
}
AppConfig.INSTANCE.get("db.url");

// When enum isn't possible — Double-Checked Locking (DCL) with volatile
public class ConnectionPool {
    private static volatile ConnectionPool instance;        // volatile is REQUIRED — see [01 §5.1]
    private ConnectionPool() {}
    public static ConnectionPool getInstance() {
        ConnectionPool local = instance;
        if (local == null) {
            synchronized (ConnectionPool.class) {
                local = instance;
                if (local == null) instance = local = new ConnectionPool();
            }
        }
        return local;
    }
}
```

> ⚠ Singletons are global state. They're hard to test (you can't inject a mock), they outlive method scope, and they couple unrelated code through their shared instance. **In a Spring app, prefer a Spring-managed bean (default scope is singleton anyway) injected via DI** — you get the same one-instance guarantee plus testability.

### 4.2 Factory Method and Abstract Factory

**Factory Method** — a method that decides which subclass to instantiate based on input.

```java
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SmsNotification();
            case "PUSH"  -> new PushNotification();
            default      -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}
```

**Abstract Factory** — an interface for creating *families* of related objects.

```java
public interface UIFactory {
    Button   createButton();
    Checkbox createCheckbox();
}
public class DarkThemeFactory  implements UIFactory { /* dark variants */ }
public class LightThemeFactory implements UIFactory { /* light variants */ }
```

**In the wild:** `Calendar.getInstance()`, `NumberFormat.getInstance()`, `DriverManager.getConnection()`, every Spring `@Bean` method.

### 4.3 Builder — complex construction step by step

**Intent:** separate the construction of a complex object from its representation, especially when there are many optional parameters.

```java
public class TradeOrder {
    private final String symbol;       // required
    private final int quantity;        // required
    private final BigDecimal price;    // optional
    private final String notes;        // optional

    private TradeOrder(Builder b) {
        this.symbol = b.symbol; this.quantity = b.quantity;
        this.price  = b.price;  this.notes    = b.notes;
    }

    public static class Builder {
        private final String symbol;
        private final int quantity;
        private BigDecimal price;
        private String notes;
        public Builder(String symbol, int quantity) {
            this.symbol = symbol; this.quantity = quantity;
        }
        public Builder price(BigDecimal p) { this.price = p; return this; }    // fluent
        public Builder notes(String n)     { this.notes = n; return this; }
        public TradeOrder build() {
            // validate here
            return new TradeOrder(this);
        }
    }
}

TradeOrder o = new TradeOrder.Builder("AAPL", 100)
    .price(new BigDecimal("150.50"))
    .notes("Market order")
    .build();
```

> 💡 Lombok's `@Builder` generates this for you. Java 16+ records *don't* eliminate the use case — records have a single canonical constructor; Builder is still useful for many optional parameters.

### 4.4 Strategy — swap algorithms at runtime

**Intent:** define a family of interchangeable algorithms, encapsulated behind a common interface.

```java
public interface PricingStrategy { BigDecimal price(Trade t); }
public class StandardPricing implements PricingStrategy { /* … */ }
public class VipPricing      implements PricingStrategy { /* … */ }

public class TradingEngine {
    private final PricingStrategy pricing;                // injected — swappable
    public TradingEngine(PricingStrategy pricing) { this.pricing = pricing; }
    public BigDecimal quote(Trade t) { return pricing.price(t); }
}

// Modern Java — for single-method interfaces, lambdas replace strategy classes:
public class TradingEngineLambda {
    private final Function<Trade, BigDecimal> pricingFn;
    public TradingEngineLambda(Function<Trade, BigDecimal> fn) { this.pricingFn = fn; }
}
new TradingEngineLambda(t -> t.amount().multiply(new BigDecimal("1.01")));
```

**`Comparator` IS the Strategy pattern.** So is `Predicate`, `Function`, `Runnable`, `AuthenticationProvider`.

### 4.5 Observer — one-to-many event notification

**Intent:** when one object's state changes, all dependents are notified.

```java
// Spring's ApplicationEvent — Observer baked in
public class TradeExecutedEvent extends ApplicationEvent {
    private final Trade trade;
    public TradeExecutedEvent(Object source, Trade trade) { super(source); this.trade = trade; }
    public Trade getTrade() { return trade; }
}

@Service
public class TradingService {
    @Autowired private ApplicationEventPublisher publisher;
    public void execute(Trade t) {
        // ... execute ...
        publisher.publishEvent(new TradeExecutedEvent(this, t));
    }
}

@Component
public class AuditListener {
    @EventListener public void onExec(TradeExecutedEvent e) { auditLog.record(e.getTrade()); }
}

@Component
public class SettlementListener {
    @EventListener public void onExec(TradeExecutedEvent e) { settlement.queue(e.getTrade()); }
}
// Multiple listeners react independently — fully decoupled from TradingService.
```

> 💡 Spring's listener model handles thread context (sync by default, `@Async` for asynchronous) and ordering (`@Order`) — all the ceremony of a hand-rolled Observer disappears.

### 4.6 Proxy — control access or add cross-cutting behaviour

**Intent:** provide a stand-in for another object to control access or add behaviour around its calls.

```java
@Service
public class PaymentService {
    @Transactional                                   // ← Spring wraps this class in a CGLIB proxy
    public void pay(Payment p) {
        // The proxy starts a transaction BEFORE the call,
        // commits AFTER (or rolls back on RuntimeException).
        repository.save(p);
    }
}
```

The caller calls `paymentService.pay(p)`. They don't see the proxy. The proxy intercepts, opens the transaction, calls the real method, commits.

> ⚠ **Self-invocation skips the proxy.** Calls via `this.method()` go directly to the implementation, bypassing all proxy logic:

```java
@Service
public class OrderService {
    @Transactional public void placeOrder(Order o) { /* … */ }
    public void batchProcess(List<Order> orders) {
        for (Order o : orders) {
            this.placeOrder(o);            // ❌ NO transaction — proxy bypassed
        }
    }
}
```

Fix: inject self via `@Autowired`, use `TransactionTemplate`, or restructure so the call crosses a proxy boundary. See [14 Spring AOP & Data](./14_spring_aop_data.md).

### 4.7 Decorator — add features by wrapping

**Intent:** attach additional behaviour to an object dynamically by wrapping it in another object that implements the same interface.

```java
// Java I/O is the canonical Decorator chain:
InputStream raw      = new FileInputStream("data.csv");      // raw bytes
InputStream buffered = new BufferedInputStream(raw);         // + buffering
DataInputStream data = new DataInputStream(buffered);        // + typed reads
// Each wrapper IS-A InputStream, HAS-A InputStream, adds one capability.

// Custom decorator
public interface TradeValidator { boolean valid(Trade t); }
public class BaseValidator implements TradeValidator { /* core checks */ }
public class LoggingValidator implements TradeValidator {
    private final TradeValidator inner;
    public LoggingValidator(TradeValidator inner) { this.inner = inner; }
    public boolean valid(Trade t) {
        log.info("validating: {}", t);
        boolean r = inner.valid(t);
        log.info("result: {}", r);
        return r;
    }
}
TradeValidator chain = new LoggingValidator(new BaseValidator());
```

Decorator and Proxy look similar in code — the difference is intent. **Proxy controls access (or adds invisible cross-cutting concerns).** **Decorator adds visible new capabilities** that callers may opt in or out of.

### 4.8 Adapter — bridge incompatible interfaces

**Intent:** convert one interface to another that the client expects.

```java
// You expect TradeProvider; the legacy system returns XmlResponse
public interface TradeProvider { List<Trade> getTrades(LocalDate date); }

public class LegacyTradeAdapter implements TradeProvider {
    private final LegacyXmlTradeService legacy;             // adaptee
    public LegacyTradeAdapter(LegacyXmlTradeService legacy) { this.legacy = legacy; }

    @Override public List<Trade> getTrades(LocalDate date) {
        XmlResponse xml = legacy.fetchTrades(date.toString());
        return xml.records().stream().map(this::toTrade).toList();
    }
    private Trade toTrade(XmlRecord r) { /* … */ }
}
```

`Arrays.asList(arr)` is a tiny adapter — turns a `T[]` into a `List<T>` view. Spring's `HandlerAdapter` adapts varied controller types (annotation-based, functional, legacy) to the common `DispatcherServlet` invocation contract.

### 4.9 Template Method — algorithm skeleton with pluggable steps

**Intent:** define the skeleton of an algorithm in a base class, defer specific steps to subclasses.

```java
public abstract class ReportGenerator {
    // The TEMPLATE METHOD — public final, defines the algorithm.
    public final Report generate() {
        var data = fetchData();              // step 1
        var formatted = format(data);        // step 2
        validate(formatted);                 // step 3 — shared default, subclass may override
        return export(formatted);            // step 4
    }
    protected abstract Object fetchData();
    protected abstract Object format(Object data);
    protected abstract Report export(Object formatted);
    protected void validate(Object data) {
        if (data == null) throw new IllegalStateException();
    }
}

public class PdfReport extends ReportGenerator {
    protected Object fetchData()                 { return db.query("..."); }
    protected Object format(Object data)         { return formatAsPdf(data); }
    protected Report export(Object formatted)    { return savePdf(formatted); }
}
```

**`HttpServlet` is the Template Method pattern:** `service()` is the template, dispatching to `doGet`, `doPost`, etc.

---

## 5. Going Deep — Interview-Level Material

### 5.1 When patterns are over-engineering

A pattern is overhead — extra interface, extra class, extra indirection. Worth the cost only when:

- The variation is **real**, not hypothetical. Two implementations actually exist (or will, on a known timeline).
- The **swap point** is genuine. If you'll never have a different `OrderRepository`, the interface is noise.
- The team understands the pattern. An unrecognised pattern looks like ceremony.

A common anti-pattern: introducing Strategy or Factory at the first sign of an `if/else`. Three branches and a stable list of known cases is *not* a Strategy candidate — a switch expression is clearer. Reach for the pattern when (a) the list grows, (b) the branches need to be added by other modules, or (c) testing requires injection.

### 5.2 How Spring uses the patterns internally

| Pattern | Where in Spring |
|---|---|
| **Singleton** | Default bean scope. One instance per `ApplicationContext`. |
| **Factory** | `BeanFactory` — every `@Bean` method *is* a factory. `FactoryBean<T>` interface for advanced cases. |
| **Builder** | `WebClient.builder()`, `MockMvcBuilders`, fluent test setup. |
| **Strategy** | `AuthenticationProvider`, `MessageConverter`, `HandlerInterceptor`. |
| **Observer** | `ApplicationEventPublisher` + `@EventListener`. |
| **Proxy** | All AOP-driven concerns: `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`. |
| **Decorator** | `WebRequest`/`HandlerWrapper` patterns; `Filter` chains. |
| **Adapter** | `HandlerAdapter` (annotated controllers / functional / legacy → unified invocation). |
| **Template Method** | `JdbcTemplate`, `RedisTemplate`, `RestTemplate` (now `RestClient`/`WebClient`), `JmsTemplate`. |

### 5.3 Modern Java replacements for old pattern boilerplate

A surprising number of GoF patterns shrink to nearly nothing in modern Java:

| Pattern | Modern shortcut |
|---|---|
| Strategy with single-method interface | A lambda. `Comparator.comparing(Employee::name)`. |
| Builder | `record` + compact constructor for the simple cases; Lombok `@Builder` for the full thing. |
| Observer | Spring `@EventListener` (or Reactor `Flux`/`Mono` for streams). |
| Singleton | `enum Singleton { INSTANCE; }` or a Spring bean. |
| Factory | `Supplier<T>` + lambda; or a Spring `@Bean` method. |
| Iterator | `Iterable.forEach`, streams, enhanced for-loops. |

The patterns themselves haven't gone away — the *boilerplate* of expressing them has.

### 5.4 Patterns that combine

Real systems compose patterns:

- **Spring `JdbcTemplate`** — Template Method for the algorithm skeleton (acquire connection → execute → release), Strategy injected as the lambda doing the actual SQL work. Often invoked through a Decorator (transactions) and a Proxy.
- **Java I/O** — Decorator chains (FileInputStream → BufferedInputStream → DataInputStream), often built by a Factory (`Files.newBufferedReader`).
- **Servlet filters** — Chain of Responsibility (each filter calls `chain.doFilter` to delegate) which is essentially a runtime Decorator pipeline.

When you read complex modern Java code, **identifying the patterns is what makes it readable**. "Oh, that's a strategy." "That's a decorator chain." "That's a proxy adding cross-cutting concerns." The vocabulary collapses an apparent maze into recognisable shapes.

---

## 6. Memory Aids

### Decision tree: "I have an `if/else` chain on type — what now?"

```
How many branches and how often will the list change?
├── 2-3, stable list, all in the same package → keep the if/else (or switch expression).
├── Many, or list grows from outside this module → STRATEGY.
└── The branches CONSTRUCT different objects, not just choose behaviour → FACTORY.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "How would you implement Singleton?" | enum or DCL | "Best is `enum INSTANCE`. If not possible, double-checked locking with volatile. Better in Spring: a default-scope bean." |
| "Strategy vs Template Method?" | Composition vs inheritance | "Strategy injects an algorithm; Template Method has the parent define the skeleton and subclasses override steps." |
| "Decorator vs Proxy?" | Visible new capability vs invisible control | "Both wrap and delegate. Decorator is for capabilities the caller wants; Proxy is for cross-cutting concerns the caller shouldn't see." |
| "Why does `@Transactional` on a self-call not work?" | Proxy bypass | "Self-invocation calls the impl directly. The proxy isn't in the path." |
| "Builder vs telescoping constructors?" | Optional params, validation | "Builder when you have many optional params or a validation step before construction." |
| "Observer vs `Flux`?" | Broadcast vs reactive stream | "Observer is one-shot per event. Flux is a stream — backpressure, composition, async." |

### Three anchor pictures

1. **Strategy is a value.** Pass the algorithm in. Comparator is the textbook example.
2. **Proxy controls access; Decorator adds capability.** Both wrap.
3. **Spring's machinery is patterns all the way down.** Beans are factories; transactions are proxies; events are observers.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: Singleton — how to implement correctly?
**A:** Best: `enum Singleton { INSTANCE; }` — thread-safe, serialisation-safe, simple. Alternative: double-checked locking with `volatile`. In Spring, prefer a managed bean (default scope is singleton). Singletons are global state — hard to test, so use sparingly outside of frameworks.

### Q2: Factory Method vs Abstract Factory?
**A:** Factory Method: a single method picks which subclass to instantiate (`Notification.create(type)`). Abstract Factory: an interface for creating *families* of related objects, ensuring consistency (e.g. `UIFactory` produces `Button` + `Checkbox` of the same theme).

### Q3: When use Builder?
**A:** Many constructor parameters (>4), especially optional. Telescoping constructors become unreadable; Builder is fluent and easy to extend. Lombok `@Builder` generates it.

### Q4: Strategy — explain it.
**A:** Define a family of algorithms, encapsulate each behind a common interface, make them interchangeable. Client picks the algorithm at runtime. In modern Java, lambdas often replace whole strategy classes. `Comparator` is the canonical example.

### Q5: Observer — explain it.
**A:** One-to-many dependency. Subject notifies observers when its state changes. Decouples producers from consumers. Spring's `ApplicationEventPublisher` + `@EventListener` is built-in Observer.

### Q6: Proxy — explain it.
**A:** Stand-in object that controls access to the real one. Spring uses CGLIB / JDK proxies for `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`. The caller calls the proxy without realising it.

### Q7: Why doesn't `@Transactional` work on self-invocation?
**A:** Self-invocation calls go directly to the implementation, bypassing the proxy. Cross-cutting concerns are added by the proxy, so they're skipped. Fix: inject self, use `TransactionTemplate`, or restructure.

### Q8: Decorator — explain it.
**A:** Wraps an object to add behaviour without modifying it. Same interface as the inner. Java I/O is built on Decorator: `new BufferedReader(new InputStreamReader(new FileInputStream("file")))`.

### Q9: Decorator vs Proxy — what's the difference?
**A:** Both wrap-and-delegate with the same interface. **Proxy controls access** (hidden cross-cutting, like transaction management). **Decorator adds capability** the caller knows about (buffering, logging, encryption).

### Q10: Adapter — explain it.
**A:** Converts one interface to another the client expects. Common at integration layers — e.g. wrapping a legacy XML service to satisfy a domain `TradeProvider` interface.

### Q11: Template Method — explain it.
**A:** Parent defines the skeleton of an algorithm in a `final` template method that calls abstract or default-implementation steps. Subclasses fill in the steps. `HttpServlet.service()` is the canonical example.

### Q12: How does Spring `JdbcTemplate` combine patterns?
**A:** Template Method (skeleton: get connection → execute → release) with Strategy injected as a lambda (the actual SQL/PreparedStatement work). Often invoked through a `@Transactional` Proxy.

### Q13: Modern Java replacements for patterns?
**A:** Lambda for single-method Strategy. `Supplier<T>` for Factory. Spring `@EventListener` for Observer. `enum INSTANCE` for Singleton. The patterns still apply; the boilerplate shrinks.

### Q14: Why is `ConnectionPool.getInstance()` with DCL using `volatile`?
**A:** `new ConnectionPool()` isn't atomic — allocate memory, write fields, publish reference. Without `volatile`, the JIT can reorder these so another thread sees a non-null reference to a partially-constructed object. `volatile` forbids that reordering. See [01 §5.1](./01_concurrency.md#51-the-java-memory-model-and-happens-before).

### Q15: When are patterns over-engineering?
**A:** When the variation is hypothetical (one implementation, no near-term plan for another), the swap point is fake (the abstraction will never be replaced), or the team doesn't recognise the pattern (introduces friction without payoff). Three branches that don't grow → just write a switch expression.

### Q16: Name an example of each from the JDK or Spring.
**A:**
- Singleton — `Runtime.getRuntime()`, default-scope Spring beans.
- Factory — `Calendar.getInstance()`, every `@Bean` method.
- Builder — `StringBuilder`, `Stream.builder()`, `WebClient.builder()`.
- Strategy — `Comparator`, `Predicate`, `AuthenticationProvider`.
- Observer — `ApplicationEventPublisher`, `PropertyChangeListener`.
- Proxy — Spring AOP (`@Transactional`, `@Cacheable`).
- Decorator — Java I/O streams, `Collections.unmodifiableList`.
- Adapter — `Arrays.asList`, Spring `HandlerAdapter`.
- Template Method — `HttpServlet`, `JdbcTemplate`, `AbstractList`.

---

### Key Code Patterns

**Enum singleton**
```java
public enum AppConfig { INSTANCE; public String get(String k) { /* … */ } }
```

**Strategy via lambda**
```java
employees.sort(Comparator.comparing(Employee::name));
new TradingEngine((Trade t) -> t.amount().multiply(BigDecimal.ONE));
```

**Builder**
```java
TradeOrder o = new TradeOrder.Builder("AAPL", 100).price(p).notes(n).build();
```

**Spring Observer**
```java
@EventListener void on(TradeExecutedEvent e) { /* react */ }
```

**JDK Decorator chain**
```java
new BufferedReader(new InputStreamReader(new FileInputStream("a")))
```

---

## 8. Self-Test

**Easy**
- [ ] What problem does Strategy solve?
- [ ] How does Spring use Proxy?
- [ ] Why is `enum Singleton` preferred?
- [ ] What's the difference between Decorator and Proxy in *intent*?

**Medium**
- [ ] Write a Strategy implementation using a lambda for sorting.
- [ ] Why must the singleton field in DCL be `volatile`?
- [ ] Show how `@Transactional` self-invocation breaks, and one way to fix it.
- [ ] Pick a JDK class you've used and identify the pattern it implements.

**Hard**
- [ ] Compose two patterns to solve: "rate-limit, then log, then call the real service".
- [ ] Identify three patterns inside Spring's `JdbcTemplate`.
- [ ] When is introducing Strategy worse than a switch expression?
- [ ] How does `HandlerAdapter` differ from `HandlerInterceptor` (pattern-wise)?
- [ ] Refactor a 5-branch `if/else` on type into Strategy with Spring DI.

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Singleton** | Class with exactly one instance. |
| **Factory Method** | A method that returns one of several subclass instances based on input. |
| **Abstract Factory** | An interface for creating *families* of related objects together. |
| **Builder** | Step-by-step construction of a complex object via a separate Builder class. |
| **Strategy** | An algorithm passed in as a value; swapped at runtime. |
| **Observer** | A subject notifying many observers on state change. |
| **Proxy** | A stand-in object controlling access to the real one. |
| **Decorator** | A wrapper that adds capability and delegates to the inner object. |
| **Adapter** | A wrapper that converts one interface to another. |
| **Template Method** | Parent defines the skeleton; subclasses fill in steps. |
| **Self-invocation** | A method calling another method on `this` — bypasses the proxy. |
| **Cross-cutting concern** | A behaviour that spans many classes (logging, transactions, security) — usually added via Proxy/AOP. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/08_design_patterns_doubts.md) · [← Prev: 07 JVM](./07_jvm.md) · [Next: 09 SOLID →](./09_solid.md)

[↑ Back to top](#08--design-patterns)
