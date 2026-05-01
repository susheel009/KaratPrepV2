# 09 — SOLID & Design Principles

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

---

## 🟢 Start Here — SOLID in Plain English

### What is SOLID?

SOLID is a set of **five rules** for writing clean, maintainable code. Think of them as good manners for programmers — follow them and your code is easy to change, test, and understand.

| Letter | Principle | In one sentence | Everyday analogy |
|:------:|-----------|----------------|-----------------|
| **S** | Single Responsibility | Each class does ONE thing | A chef cooks. A waiter serves. They don't swap jobs. |
| **O** | Open/Closed | Add new features without changing existing code | A phone case — add cases without redesigning the phone |
| **L** | Liskov Substitution | A subtype should work wherever the parent type is expected | If it looks like a duck and quacks like a duck, it should behave like a duck |
| **I** | Interface Segregation | Many small interfaces > one big one | A Swiss Army knife with 50 tools is unwieldy. Better: a separate knife, scissors, screwdriver. |
| **D** | Dependency Inversion | Depend on abstractions, not specifics | You depend on "a ride" (interface), not "Bob's car" (implementation). Uber, taxi, bus all work. |

### Why should you care?

Bad code **works today but breaks tomorrow**. SOLID principles help you write code that:
- Is easy to **change** when requirements change
- Is easy to **test** in isolation
- Doesn't break **other things** when you modify one part

### The simplest example

```java
// ❌ BAD: OrderService does everything (violates SRP)
class OrderService {
    void createOrder() { }
    void sendEmail() { }      // why is an order service sending emails?
    void generatePDF() { }    // why is it making PDFs?
}

// ✅ GOOD: Each class has one job
class OrderService   { void createOrder() { } }
class EmailService   { void sendEmail() { } }
class ReportService  { void generatePDF() { } }
```

> Key takeaway: SOLID = five rules to keep your code clean. S = one job per class. O = extend, don't modify. L = subtypes must behave like parents. I = small, focused interfaces. D = depend on abstractions.

---

## 📚 Study Material

### 1. Single Responsibility Principle (SRP)

**"A class should have one, and only one, reason to change."**

```java
// ❌ VIOLATION: OrderService does too much
public class OrderService {
    public void createOrder(Order order) { /* business logic */ }
    public void sendConfirmationEmail(Order order) { /* email logic */ }
    public byte[] generateInvoicePdf(Order order) { /* PDF logic */ }
    public void saveToDatabase(Order order) { /* persistence logic */ }
}
// 4 reasons to change: order rules, email format, PDF format, DB schema

// ✅ FIX: One responsibility per class
public class OrderService {
    private final OrderRepository repo;
    private final NotificationService notifications;
    
    public void createOrder(Order order) {
        validate(order);
        repo.save(order);                        // delegates persistence
        notifications.orderCreated(order);       // delegates notification
    }
}
// OrderService changes ONLY when business rules change
```

💡 **Interview tip:** "One reason to change" means one *stakeholder* or *axis of change*. The accounting team changes invoice format; the marketing team changes emails. Those are different axes → different classes.

### 2. Open/Closed Principle (OCP)

**"Open for extension, closed for modification."**

```java
// ❌ VIOLATION: Adding a new payment type requires modifying existing code
public class PaymentProcessor {
    public void process(Payment p) {
        if (p.getType() == PaymentType.CREDIT_CARD) { /* credit logic */ }
        else if (p.getType() == PaymentType.BANK_TRANSFER) { /* bank logic */ }
        else if (p.getType() == PaymentType.CRYPTO) { /* had to modify this class! */ }
    }
}

// ✅ FIX: Strategy pattern — add new types without modifying existing code
public interface PaymentHandler {
    boolean supports(PaymentType type);
    void process(Payment payment);
}

@Component public class CreditCardHandler implements PaymentHandler { /* ... */ }
@Component public class BankTransferHandler implements PaymentHandler { /* ... */ }
@Component public class CryptoHandler implements PaymentHandler { /* ... */ }  // NEW — no existing code changed

@Service
public class PaymentProcessor {
    private final List<PaymentHandler> handlers;    // Spring injects all implementations
    
    public void process(Payment p) {
        handlers.stream()
            .filter(h -> h.supports(p.getType()))
            .findFirst()
            .orElseThrow()
            .process(p);
    }
}
```

### 3. Liskov Substitution Principle (LSP)

**"Subtypes must be substitutable for their base type without breaking correctness."**

```java
// ❌ CLASSIC VIOLATION: Square extends Rectangle
public class Rectangle {
    protected int width, height;
    public void setWidth(int w)  { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override public void setWidth(int w)  { this.width = w; this.height = w; }  // breaks contract!
    @Override public void setHeight(int h) { this.width = h; this.height = h; }
}

void assertArea(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.area() == 20;     // FAILS for Square (area = 16) — LSP violated
}

// ✅ FIX: Use a Shape interface — Square and Rectangle are siblings, not parent-child
public interface Shape {
    double area();
}
public record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}
public record Square(double side) implements Shape {
    public double area() { return side * side; }
}
```

**LSP checklist:** Can every place that uses the parent type work correctly with the subtype? If no → inheritance is wrong.

### 4. Interface Segregation Principle (ISP)

**"Clients shouldn't be forced to depend on methods they don't use."**

```java
// ❌ VIOLATION: Fat interface
public interface Worker {
    void code();
    void test();
    void deploy();
    void design();
    void managePeople();  // not all workers manage people!
}

// ✅ FIX: Small, focused interfaces
public interface Coder { void code(); }
public interface Tester { void test(); }
public interface Deployer { void deploy(); }
public interface Manager { void managePeople(); }

// Classes implement only what they need
public class Developer implements Coder, Tester { /* ... */ }
public class DevOpsEngineer implements Deployer, Coder { /* ... */ }
public class TeamLead implements Coder, Manager { /* ... */ }

// JDK example: Closeable, Readable, Appendable — not one giant I/O interface
```

### 5. Dependency Inversion Principle (DIP)

**"Depend on abstractions, not concretions. High-level modules shouldn't depend on low-level modules."**

```java
// ❌ VIOLATION: High-level module depends on concrete low-level class
public class OrderService {
    private final MySQLOrderRepository repo = new MySQLOrderRepository();  // tight coupling
    // Can't test without a real MySQL database
    // Can't switch to PostgreSQL without modifying OrderService
}

// ✅ FIX: Both depend on an abstraction (interface)
public interface OrderRepository {                  // abstraction
    void save(Order order);
    Optional<Order> findById(String id);
}

@Repository
public class MySQLOrderRepository implements OrderRepository { /* ... */ }

@Service
public class OrderService {
    private final OrderRepository repo;             // depends on interface, not MySQL
    
    public OrderService(OrderRepository repo) {     // injected — easy to test with mock
        this.repo = repo;
    }
}

// Spring DI IS DIP in action:
// - High-level (OrderService) depends on interface (OrderRepository)
// - Low-level (MySQLOrderRepository) implements the interface
// - The IoC container wires them together
```

### 6. DRY, KISS, YAGNI — Complementary Principles

```java
// DRY — Don't Repeat Yourself
// Every piece of knowledge should have a SINGLE authoritative source
// Duplicated code → duplicated bugs → forgotten updates
// ⚠️ But don't over-DRY: if two pieces of code look similar but change for
//    different reasons → they should stay separate (accidental duplication ≠ real duplication)

// KISS — Keep It Simple
// If a for-loop is clearer than a stream pipeline, use the for-loop
// Don't use a design pattern just because you know it — use it when it solves a real problem

// YAGNI — You Aren't Gonna Need It
// Don't build features speculatively
// "We might need this later" → build it later, when the requirement is clear
// Speculative code has maintenance cost with zero current value
```

### Summary — SOLID at a Glance

| Principle | One Sentence | Violation Smell |
|-----------|-------------|----------------|
| **S**RP | One class, one reason to change | God class, 500+ line class |
| **O**CP | Extend without modifying | `if/else` chain for types |
| **L**SP | Subtypes must honour parent's contract | Override that changes semantics |
| **I**SP | Small, focused interfaces | Implementing interface with empty methods |
| **D**IP | Depend on interfaces, not implementations | `new ConcreteClass()` in high-level code |

---

## Rapid-Fire Q&A

### Q1: Single Responsibility Principle (SRP)?
**A:** A class should have one reason to change. One responsibility, one axis of change. E.g., `OrderService` handles order logic — not email sending, not PDF generation. Violation: God class that does everything.

### Q2: Open/Closed Principle (OCP)?
**A:** Open for extension, closed for modification. Add new behaviour by adding new classes (implementing an interface), not modifying existing code. Strategy pattern enables this.

### Q3: Liskov Substitution Principle (LSP)?
**A:** Subtypes must be substitutable for base types without breaking correctness. Classic violation: `Square extends Rectangle` — setting width on a Square also changes height, breaking Rectangle's contract.

### Q4: Interface Segregation Principle (ISP)?
**A:** Many small interfaces > one fat interface. Clients shouldn't be forced to depend on methods they don't use. E.g., `Readable`, `Writable`, `Closeable` — not one giant `FileHandler` interface.

### Q5: Dependency Inversion Principle (DIP)?
**A:** Depend on abstractions, not concretions. High-level modules shouldn't depend on low-level modules — both should depend on interfaces. Spring DI is DIP in action: service depends on `PaymentGateway` interface, not `StripePaymentGateway` class.

### Q6: DRY — Don't Repeat Yourself?
**A:** Every piece of knowledge has a single source of truth. Duplicated code → duplicated bugs. Extract common logic into shared methods/classes. But: don't over-DRY — premature abstraction is worse than duplication.

### Q7: KISS — Keep It Simple?
**A:** Simpler solutions are better. If a problem can be solved with a `for` loop, don't use a stream pipeline with 6 stages. Complexity is the enemy of maintainability.

### Q8: YAGNI — You Aren't Gonna Need It?
**A:** Don't build features speculatively. Build what's needed now, refactor later when requirements are clear. Over-engineering is a form of waste.

---

## Can you answer these cold?

- [ ] All five SOLID principles — one sentence each
- [ ] Give a violation example for LSP
- [ ] DIP — how Spring DI implements it
- [ ] When DRY goes too far (premature abstraction)

[← Back to Index](./00_INDEX.md)
