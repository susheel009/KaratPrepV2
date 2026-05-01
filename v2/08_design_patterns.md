# 08 — Design Patterns

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

---

## 🟢 Start Here — Design Patterns in Plain English

### What are design patterns?

Design patterns are **proven solutions to common problems**. They're like recipes — you don't invent a new way to make pasta every time. Someone figured out the best way, wrote it down, and now everyone uses it.

### The patterns you need to know (in one sentence each)

| Pattern | One-sentence explanation | Real-life analogy |
|---------|------------------------|-------------------|
| **Singleton** | Only one instance of a class exists in your entire app | There's only one president at a time |
| **Factory** | A method creates the right object for you based on input | A pizza shop: you say "pepperoni" → they make the right pizza |
| **Builder** | Build a complex object step by step | Ordering a custom sandwich: bread → meat → cheese → sauce → done |
| **Strategy** | Swap out different algorithms without changing the rest | GPS navigation: pick "fastest route", "shortest", or "avoid tolls" — same destination, different approach |
| **Observer** | When something changes, notify everyone interested | YouTube subscriptions: when a channel uploads, all subscribers get notified |
| **Proxy** | A stand-in that controls access to the real thing | A receptionist who screens calls before putting you through to the boss |
| **Decorator** | Add features by wrapping objects | Plain coffee → add milk (wrapper) → add sugar (wrapper) → add whipped cream (wrapper) |
| **Adapter** | Make incompatible things work together | A power plug adapter when you travel abroad |
| **Template Method** | Define the recipe steps, let subclasses fill in the details | A job application form: the structure is fixed (name, experience, education) but each person fills it in differently |

### Why do they matter?

Without patterns, every developer invents their own way to solve the same problem — leading to messy, inconsistent code that's hard for others to understand. Patterns give you a **shared vocabulary**: when you say "we used the Strategy pattern," everyone knows exactly what you mean.

> Key takeaway: Design patterns are named solutions to common problems. You don't need to memorise every detail — understand the *intent* (what problem it solves) and *when to use it*.

---

## 📚 Study Material

### 1. Singleton — One Instance, Global Access

**Intent:** Ensure a class has exactly one instance with a global access point.

```java
// BEST: Enum singleton — thread-safe, serialisation-safe, simple
public enum AppConfig {
    INSTANCE;
    private final Properties props = loadProps();
    public String get(String key) { return props.getProperty(key); }
}
AppConfig.INSTANCE.get("db.url");

// Double-Checked Locking (when enum isn't possible, e.g., inheritance)
public class ConnectionPool {
    private static volatile ConnectionPool instance;    // volatile prevents partial construction
    private ConnectionPool() { /* init pool */ }
    
    public static ConnectionPool getInstance() {
        if (instance == null) {                         // 1st check — no lock (fast path)
            synchronized (ConnectionPool.class) {
                if (instance == null) {                 // 2nd check — under lock
                    instance = new ConnectionPool();
                }
            }
        }
        return instance;
    }
}

// ⚠️ Singleton = global state → hard to test (can't inject mock)
// Prefer: dependency injection (Spring manages singletons for you)
```

### 2. Factory Method — Decouple Creation

**Intent:** Define an interface for creating objects, let subclasses decide which class to instantiate.

```java
// Simple factory (most common in practice)
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SmsNotification();
            case "PUSH"  -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}

// Abstract Factory — family of related objects
public interface UIFactory {
    Button createButton();       // each factory creates a consistent family
    Checkbox createCheckbox();
}
public class DarkThemeFactory implements UIFactory {
    public Button createButton() { return new DarkButton(); }
    public Checkbox createCheckbox() { return new DarkCheckbox(); }
}
// Use when: you need to create groups of related objects that must be consistent
```

**In the wild:** `Calendar.getInstance()`, `NumberFormat.getInstance()`, `DriverManager.getConnection()`.

### 3. Builder — Complex Object Construction

**Intent:** Separate construction of a complex object from its representation.

```java
public class TradeOrder {
    private final String symbol;       // required
    private final int quantity;        // required
    private final BigDecimal price;    // optional
    private final String notes;        // optional
    
    private TradeOrder(Builder b) {
        this.symbol = b.symbol;
        this.quantity = b.quantity;
        this.price = b.price;
        this.notes = b.notes;
    }
    
    public static class Builder {
        private final String symbol;           // required — in constructor
        private final int quantity;            // required — in constructor
        private BigDecimal price;              // optional — via setter
        private String notes;                  // optional — via setter
        
        public Builder(String symbol, int quantity) {
            this.symbol = symbol;
            this.quantity = quantity;
        }
        public Builder price(BigDecimal p)  { this.price = p; return this; }   // fluent
        public Builder notes(String n)      { this.notes = n; return this; }
        
        public TradeOrder build() {
            // validate here
            return new TradeOrder(this);
        }
    }
}

TradeOrder order = new TradeOrder.Builder("AAPL", 100)
    .price(new BigDecimal("150.50"))
    .notes("Market order")
    .build();

// Lombok shortcut: @Builder on the class generates all of this
```

### 4. Strategy — Swap Algorithms at Runtime

**Intent:** Define a family of algorithms, encapsulate each, make them interchangeable.

```java
// Traditional approach
public interface PricingStrategy {
    BigDecimal calculatePrice(Trade trade);
}
public class StandardPricing implements PricingStrategy { /* ... */ }
public class VIPPricing implements PricingStrategy { /* ... */ }

public class TradingEngine {
    private final PricingStrategy pricing;                    // injected — swappable
    public TradingEngine(PricingStrategy pricing) { this.pricing = pricing; }
    public BigDecimal quote(Trade trade) { return pricing.calculatePrice(trade); }
}

// Modern Java — lambdas replace single-method strategy classes
public class TradingEngine {
    private final Function<Trade, BigDecimal> pricingFn;     // lambda = strategy
    public TradingEngine(Function<Trade, BigDecimal> pricingFn) { this.pricingFn = pricingFn; }
}
new TradingEngine(trade -> trade.getAmount().multiply(new BigDecimal("1.01")));  // inline strategy

// JDK example: Comparator IS the strategy pattern
list.sort(Comparator.comparing(Employee::getName));          // strategy as lambda
list.sort(Comparator.comparing(Employee::getSalary).reversed());
```

### 5. Observer — Event-Driven Notification

**Intent:** One-to-many dependency — when one object changes, all dependents are notified.

```java
// Spring's ApplicationEvent — observer pattern built-in
public class TradeExecutedEvent extends ApplicationEvent {
    private final Trade trade;
    public TradeExecutedEvent(Object source, Trade trade) {
        super(source);
        this.trade = trade;
    }
}

// Publisher (subject)
@Service
public class TradingService {
    @Autowired private ApplicationEventPublisher publisher;
    
    public void execute(Trade trade) {
        // ... execute trade ...
        publisher.publishEvent(new TradeExecutedEvent(this, trade));  // notify observers
    }
}

// Observer (listener) — decoupled, doesn't know about TradingService
@Component
public class AuditListener {
    @EventListener
    public void onTradeExecuted(TradeExecutedEvent event) {
        auditLog.record(event.getTrade());    // reacts to event
    }
}

// Multiple listeners can react independently — full decoupling
```

### 6. Proxy — Control Access

**Intent:** Provide a surrogate for another object to control access.

```java
// Spring uses proxies for @Transactional, @Cacheable, @Async, @Secured
// You call → Proxy intercepts → adds behaviour → delegates to real object

@Service
public class PaymentService {
    @Transactional                         // Spring wraps this class in a CGLIB proxy
    public void processPayment(Payment p) {
        // Proxy starts transaction BEFORE this method
        repository.save(p);
        // Proxy commits transaction AFTER this method (or rolls back on exception)
    }
}

// ⚠️ Self-invocation bypasses the proxy:
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order o) { /* has transaction */ }
    
    public void batchProcess(List<Order> orders) {
        for (Order o : orders) {
            this.placeOrder(o);            // ❌ calls directly — NO proxy, NO transaction
        }
    }
}
// Fix: inject self, or use TransactionTemplate, or restructure
```

### 7. Decorator — Add Behaviour by Wrapping

**Intent:** Attach additional responsibilities dynamically by wrapping.

```java
// Java I/O — classic decorator chain
InputStream raw = new FileInputStream("data.csv");          // raw bytes
InputStream buffered = new BufferedInputStream(raw);         // + buffering
InputStream data = new DataInputStream(buffered);           // + typed reading
// Each wrapper adds a layer without modifying the inner stream

// Custom decorator
public interface TradeValidator {
    boolean validate(Trade trade);
}
public class BaseValidator implements TradeValidator { /* core validation */ }
public class LoggingValidator implements TradeValidator {
    private final TradeValidator inner;
    public LoggingValidator(TradeValidator inner) { this.inner = inner; }
    public boolean validate(Trade trade) {
        log.info("Validating: {}", trade);
        boolean result = inner.validate(trade);              // delegate
        log.info("Result: {}", result);
        return result;
    }
}
// new LoggingValidator(new BaseValidator()) — added logging without modifying BaseValidator
```

### 8. Adapter — Bridge Incompatible Interfaces

**Intent:** Convert one interface to another that the client expects.

```java
// Legacy system returns XML, your code expects domain objects
public class LegacyTradeSystemAdapter implements TradeProvider {
    private final LegacyXmlTradeService legacy;   // adaptee (old interface)
    
    @Override
    public List<Trade> getTrades(LocalDate date) {
        XmlResponse xml = legacy.fetchTrades(date.toString());  // call old system
        return xml.getRecords().stream()
            .map(this::convertXmlToTrade)                       // convert to new model
            .toList();
    }
}
// Client code uses TradeProvider interface — doesn't know about legacy XML system
```

### 9. Template Method — Algorithm Skeleton

**Intent:** Define the skeleton of an algorithm; subclasses fill in steps.

```java
public abstract class ReportGenerator {
    // Template method — defines the algorithm
    public final Report generate() {
        var data = fetchData();             // step 1: subclass provides
        var formatted = format(data);       // step 2: subclass provides
        validate(formatted);                // step 3: shared implementation
        return export(formatted);           // step 4: subclass provides
    }
    
    protected abstract Object fetchData();
    protected abstract Object format(Object data);
    protected abstract Report export(Object formatted);
    
    protected void validate(Object data) {   // default implementation — can be overridden
        if (data == null) throw new IllegalStateException("No data");
    }
}

public class PdfReport extends ReportGenerator {
    protected Object fetchData() { return db.query("..."); }
    protected Object format(Object data) { return formatAsPdf(data); }
    protected Report export(Object formatted) { return savePdf(formatted); }
}
```

### Quick Reference — When to Use Each

| Pattern | When | JDK/Spring Example |
|---------|------|-------------------|
| Singleton | Exactly one instance needed | `Runtime.getRuntime()`, Spring beans (default scope) |
| Factory | Decouple creation from usage | `Calendar.getInstance()`, `DriverManager.getConnection()` |
| Builder | Many constructor parameters | `StringBuilder`, `Stream.Builder`, Lombok `@Builder` |
| Strategy | Swap algorithms at runtime | `Comparator`, `Predicate`, Spring `AuthenticationProvider` |
| Observer | React to events (decouple) | `ApplicationEventPublisher`, `PropertyChangeListener` |
| Proxy | Add cross-cutting behaviour | Spring AOP, `@Transactional`, `@Cacheable` |
| Decorator | Layer additional behaviour | Java I/O streams, `Collections.unmodifiableList()` |
| Adapter | Bridge old and new interfaces | `Arrays.asList()`, Spring `HandlerAdapter` |
| Template Method | Algorithm skeleton + custom steps | `AbstractList`, `HttpServlet.doGet()` |

---

## Rapid-Fire Q&A

### Q1: Singleton — how to implement correctly?
**A:** Best: `enum Singleton { INSTANCE; }` — serialisation-safe, thread-safe, simple. Alternative: double-checked locking with `volatile`. Avoid: eager init with `static final` (okay but wastes memory if unused). Anti-pattern warning: singletons are global state — hard to test.

### Q2: Factory Method vs Abstract Factory?
**A:** Factory Method: single method decides which subclass to instantiate (`PaymentProcessor.create(type)`). Abstract Factory: family of related objects (`UIFactory.createButton()`, `UIFactory.createCheckbox()` — each factory produces consistent platform UI).

### Q3: Builder — when use it?
**A:** When constructor has many parameters (>4), especially optional ones. Fluent API: `Account.builder().name("Alice").balance(1000).build()`. Avoids telescoping constructors. Lombok `@Builder` auto-generates.

### Q4: Strategy — explain it.
**A:** Define a family of algorithms, encapsulate each, make them interchangeable. E.g., `SortStrategy` → `QuickSort`, `MergeSort`. Client picks strategy at runtime. In modern Java, lambdas often replace strategy classes: `list.sort(Comparator.comparing(Employee::getName))`.

### Q5: Observer — explain it.
**A:** One-to-many dependency. Subject notifies observers on state change. Event-driven systems, pub-sub. Spring's `ApplicationEventPublisher` is an observer pattern. Decouples producers from consumers.

### Q6: Proxy — explain it.
**A:** Control access to another object. Spring AOP uses CGLIB proxies for `@Transactional`, `@Cacheable`, `@Async`. Also: lazy init proxy, security proxy, remote proxy. The caller doesn't know it's talking to a proxy.

### Q7: Template Method — explain it.
**A:** Superclass defines algorithm skeleton, subclasses override specific steps. E.g., `AbstractReport.generate()` calls `fetchData()`, `format()`, `export()` — subclasses override each. In Java 8+, often replaced with lambdas or strategy.

### Q8: Decorator — explain it.
**A:** Wraps an object to add behaviour without modifying it. Java I/O uses it: `new BufferedReader(new InputStreamReader(new FileInputStream("file")))`. Each wrapper adds a layer. Open/Closed principle in action.

### Q9: Adapter — explain it.
**A:** Converts one interface to another. Legacy system returns `XmlResponse`; your code expects `JsonResponse` — adapter converts between them. Common in integration layers.

---

## Can you answer these cold?

- [ ] Singleton — enum vs DCL vs eager init
- [ ] Factory vs Builder — when each
- [ ] Strategy — modern Java lambda version
- [ ] Proxy — how Spring uses it for AOP
- [ ] Decorator — Java I/O example

[← Back to Index](./00_INDEX.md)
