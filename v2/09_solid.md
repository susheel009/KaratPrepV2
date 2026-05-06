# 09 — SOLID & Design Principles

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/09_solid_doubts.md) · [← Prev: 08 Design Patterns](./08_design_patterns.md) · [Next: 10 Testing →](./10_testing.md)
>
> **Priority:** 🟢 Medium · **Related topics:** [08 Design Patterns](./08_design_patterns.md) · [03 OOP (composition vs inheritance)](./03_oop_language.md) · [11 Spring Core (DI = DIP)](./11_spring_core.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — five rules for code that ages well
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — refactoring a god class
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — bad → good SRP
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 SRP](#41-srp--single-responsibility)
   - [4.2 OCP](#42-ocp--open-for-extension-closed-for-modification)
   - [4.3 LSP](#43-lsp--liskov-substitution)
   - [4.4 ISP](#44-isp--interface-segregation)
   - [4.5 DIP](#45-dip--dependency-inversion)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 SOLID is a starting point, not gospel](#51-solid-is-a-starting-point-not-gospel)
   - [5.2 DRY, KISS, YAGNI](#52-dry-kiss-yagni--complementary-rules)
   - [5.3 Composition vs inheritance](#53-composition-vs-inheritance--cross-link-to-03)
   - [5.4 SOLID and Spring DI](#54-solid-and-spring-di-dip-in-action)
   - [5.5 Common violations to spot in code review](#55-common-violations-to-spot-in-code-review)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

You're a building inspector visiting two warehouses.

Warehouse A is a single huge open hall. Forklifts share the floor with packers, the loading bay opens directly into the office, the toilets are next to the food prep, and the office printer is on the same circuit as the freezer. When the freezer trips the breaker, the printer dies. When the loading bay floods, the office floods too. Adding anything new — a new product line, a new shipping zone — means rearranging *everything*, because *everything is everywhere*.

Warehouse B is the same square footage, but it's been thoughtfully zoned. Receiving has its own dock and conditioning. Cold storage is its own walled-off room with its own circuit. Office, washrooms, and shipping each have their own walls. Adding a new product line means **adding a new zone** without touching the existing ones.

Same square footage, completely different cost-of-change. The inspector approves Warehouse B because *changes don't ripple*.

That's what SOLID does for code. Five small rules from Robert C. Martin (mid-2000s) that, when followed honestly, give you a Warehouse B instead of a Warehouse A:

| Letter | One-line | Warehouse analogue |
|:--:|---|---|
| **S** | Single responsibility — each class has one reason to change | One zone per purpose |
| **O** | Open for extension, closed for modification | Add a new room, don't gut existing rooms |
| **L** | Liskov substitution — subtypes must work where the base type works | A "loading dock" must actually behave like a loading dock |
| **I** | Interface segregation — many small interfaces, not one fat one | Door per purpose, not one giant warehouse-front door |
| **D** | Dependency inversion — depend on abstractions, not concretions | Office plugs into "electricity" (interface), not "this specific generator" (impl) |

> The point of SOLID isn't to make code "elegant" — it's to make change cheap. When the next requirement lands, well-zoned code lets you add or replace one room. Poorly zoned code forces you to renovate the whole hall.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

We have an `OrderService` that has grown over a year. Currently it does:

```
class OrderService:
    createOrder(...)             // business rules — the actual order creation
    sendConfirmationEmail(...)   // SMTP call, template lookup
    generateInvoicePdf(...)      // PDF library call
    saveToDatabase(...)          // SQL queries
    auditLog(...)                // log file format, file path
```

Five jobs. Five reasons to change:

1. Business rules update → modify createOrder. ✅ legitimate.
2. Email template redesigned by marketing → modify OrderService. ❌
3. PDF format changed by finance → modify OrderService. ❌
4. DB schema migrated → modify OrderService. ❌
5. Audit format changed for compliance → modify OrderService. ❌

A change to any of *those four unrelated stakeholders* forces a rebuild and redeploy of `OrderService` — and risks breaking unrelated features. The SOLID-ised version delegates each concern to its own collaborator (`NotificationService`, `InvoiceGenerator`, `OrderRepository`, `AuditLogger`) injected via the constructor. `OrderService` shrinks to just the order business rules. *It now has one reason to change.*

Two takeaways the walkthrough makes obvious:

1. **"Reason to change" means "stakeholder" or "axis of change", not "lines of code".** A 200-line class that only changes when business rules change is fine. A 50-line class that changes for four different reasons is a problem.
2. **Once each piece is in its own class, the others (OCP, LSP, ISP, DIP) become natural** — you start seeing the right interfaces, the right injection points, the right test seams.

Now the actual code, before and after.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
// === BEFORE: god class. Five reasons to change. ===
public class OrderService {
    public void createOrder(Order order) {
        // 1. Business validation
        if (order.items().isEmpty()) throw new IllegalArgumentException();
        // 2. Persistence — knows about JDBC URL
        try (var conn = DriverManager.getConnection("jdbc:..."); var stmt = conn.prepareStatement("...")) {
            // ...
        } catch (SQLException e) { throw new RuntimeException(e); }
        // 3. Notification — knows about SMTP
        var props = new Properties(); /* ... */
        Session.getInstance(props).getTransport("smtp").send(/* ... */);
        // 4. Invoice — knows about PDF library
        var doc = new PdfDocument(); /* ... */
        // 5. Audit — knows about file paths
        Files.writeString(Path.of("/var/log/orders.log"), order.toString() + "\n");
    }
}

// === AFTER: each concern in its own class. One reason to change for OrderService. ===
public class OrderService {
    private final OrderRepository repo;
    private final NotificationService notifications;
    private final InvoiceGenerator invoices;
    private final AuditLogger audit;
    public OrderService(OrderRepository repo, NotificationService n,        // constructor injection
                        InvoiceGenerator inv, AuditLogger audit) {
        this.repo = repo; this.notifications = n; this.invoices = inv; this.audit = audit;
    }
    public void createOrder(Order order) {                                  // ONLY business rules
        if (order.items().isEmpty()) throw new IllegalArgumentException();
        repo.save(order);
        notifications.orderCreated(order);
        invoices.generate(order);
        audit.recordCreate(order);
    }
}

// Each collaborator is an INTERFACE — Spring (or your test) injects an impl.
public interface OrderRepository { void save(Order order); /* ... */ }
public interface NotificationService { void orderCreated(Order order); }
public interface InvoiceGenerator { byte[] generate(Order order); }
public interface AuditLogger { void recordCreate(Order order); }
```

What just happened: every concern lives in its own class with a focused interface. `OrderService` knows nothing about JDBC, SMTP, PDF libraries, or log files — it just orchestrates the four collaborators. Marketing can rebuild the email service without rebuilding the order service. Tests can inject mocks for all four collaborators. **One class, one reason to change.**

---

## 4. Build Up — Practical Patterns

### 4.1 SRP — Single Responsibility

> *"A class should have one, and only one, reason to change."*

The classic phrasing is "one responsibility", but the operationally useful version is **one reason to change** — meaning one *stakeholder* or *axis of change*. The accounting team owns invoice format. The marketing team owns email templates. Different stakeholders → different classes.

```java
// ❌ god class
class OrderService { void createOrder(); void sendEmail(); void generatePdf(); void saveToDb(); }
// 4 reasons to change.

// ✅ split
class OrderService     { void createOrder(); }      // changes only when business rules change
class EmailService     { void sendEmail(); }        // changes only when email format changes
class InvoiceService   { void generatePdf(); }      // changes only when PDF layout changes
class OrderRepository  { void save(Order o); }      // changes only when DB schema changes
```

**SRP isn't anti-cohesion.** A 300-line class that does one thing well is fine. The cue is: *do new requirements from different stakeholders force me to edit this class?*

### 4.2 OCP — Open for extension, closed for modification

> *"Software entities should be open for extension but closed for modification."*

Add new behaviour by adding new code, not by editing existing code. The Strategy pattern enables this:

```java
// ❌ Adding a new payment type requires editing this method.
void process(Payment p) {
    if (p.type() == CREDIT) { /* ... */ }
    else if (p.type() == BANK) { /* ... */ }
    else if (p.type() == CRYPTO) { /* HAD to edit this method */ }
}

// ✅ One interface. Each handler is its own class. Adding a type means adding a new class.
interface PaymentHandler { boolean supports(PaymentType t); void process(Payment p); }
@Component class CreditCardHandler  implements PaymentHandler { /* ... */ }
@Component class BankTransferHandler implements PaymentHandler { /* ... */ }
@Component class CryptoHandler       implements PaymentHandler { /* ... */ }   // NEW — no edits

@Service
public class PaymentProcessor {
    private final List<PaymentHandler> handlers;                                // Spring injects all
    public PaymentProcessor(List<PaymentHandler> handlers) { this.handlers = handlers; }
    public void process(Payment p) {
        handlers.stream().filter(h -> h.supports(p.type())).findFirst()
            .orElseThrow().process(p);
    }
}
```

> ⚠ **OCP is aspirational, not absolute.** You can't be "closed" against changes you didn't anticipate. The pragmatic rule: when you spot the *second* `if/else` branch on the same dimension, it's time to refactor toward an OCP-friendly structure.

### 4.3 LSP — Liskov Substitution

> *"Subtypes must be substitutable for their base types without breaking correctness."*

If `S` extends `T`, every place that uses `T` must work correctly when given an `S`. The classic violation: `Square extends Rectangle`.

```java
// ❌ classic LSP violation
class Rectangle {
    int width, height;
    void setWidth(int w)  { this.width  = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}
class Square extends Rectangle {
    @Override void setWidth(int w)  { this.width  = w; this.height = w; }   // changes both!
    @Override void setHeight(int h) { this.width  = h; this.height = h; }
}
// Caller invariant: setWidth(5); setHeight(4); area() == 20
// → fails for Square (returns 16). Square breaks Rectangle's contract.

// ✅ Don't make Square a Rectangle. Make them siblings under a Shape interface.
interface Shape { double area(); }
record Rectangle(double w, double h) implements Shape { public double area() { return w * h; } }
record Square(double side)           implements Shape { public double area() { return side * side; } }
```

**LSP smells in practice:** an override that *strengthens preconditions*, *weakens postconditions*, throws unexpected exceptions, or violates a parent invariant. If you see `if (x instanceof SubType) { ... special case ... }` in callers, LSP is being violated.

### 4.4 ISP — Interface Segregation

> *"Clients should not be forced to depend on interfaces they don't use."*

A **fat interface** forces every implementor to provide methods most don't need.

```java
// ❌ fat interface
interface Worker { void code(); void test(); void deploy(); void design(); void managePeople(); }
// Now an Intern who only codes must provide empty test/deploy/design/managePeople methods.

// ✅ small, focused interfaces
interface Coder    { void code(); }
interface Tester   { void test(); }
interface Deployer { void deploy(); }
interface Manager  { void managePeople(); }

class Developer    implements Coder, Tester {}
class TeamLead     implements Coder, Manager {}
class DevOps       implements Coder, Deployer {}
```

The JDK's I/O library is a textbook example: `Closeable`, `Readable`, `Appendable`, `Flushable` instead of one mega `IOResource`.

### 4.5 DIP — Dependency Inversion

> *"High-level modules should not depend on low-level modules. Both should depend on abstractions."*

In an inverted-dependency graph, **policy** doesn't depend on **mechanism** — both depend on an abstraction defined on the policy side.

```java
// ❌ high-level OrderService depends directly on the low-level MySQL impl
public class OrderService {
    private final MySQLOrderRepository repo = new MySQLOrderRepository();   // tight coupling
    // - can't test without a real MySQL
    // - can't switch storage without modifying OrderService
}

// ✅ both depend on the OrderRepository INTERFACE, owned by the policy side
public interface OrderRepository { void save(Order order); Optional<Order> findById(String id); }

@Repository
public class MySQLOrderRepository implements OrderRepository { /* ... */ }

@Service
public class OrderService {
    private final OrderRepository repo;                    // depends on the interface
    public OrderService(OrderRepository repo) { this.repo = repo; }   // injected — easy to test
}
```

This is **literally what Spring DI is**. Spring's `ApplicationContext` (an *IoC container*) wires implementations to interfaces at startup, so the high-level code never touches a `new MySQLRepository()`. See [11 Spring Core](./11_spring_core.md).

---

## 5. Going Deep — Interview-Level Material

### 5.1 SOLID is a starting point, not gospel

SOLID is a heuristic toolkit, not a religion. Real code routinely violates pieces of it for legitimate reasons:

- **Records and POJOs intentionally violate SRP** — they bundle data + accessors + equality + hashCode + toString. That's correct: those *are* one responsibility (representing the value).
- **OCP is aspirational** — you genuinely cannot anticipate every future axis of variation. Refactor toward OCP when a *real* second axis emerges, not pre-emptively.
- **LSP is the strict one** — it's the only SOLID rule with a precise mathematical formulation, and violations almost always cause real bugs.
- **ISP is rarely violated in modern Java** — interfaces tend to stay small unless someone deliberately inflates them.
- **DIP is the most useful at scale** — without it, large codebases become impossible to test or refactor.

**Order of importance for interviews and real work, roughly:** DIP > SRP > LSP > OCP > ISP. DIP and SRP carry most of the practical weight; LSP catches concrete bugs; OCP and ISP are refinements.

### 5.2 DRY, KISS, YAGNI — complementary rules

These three sit alongside SOLID and round out the toolkit:

**DRY — Don't Repeat Yourself.** Every piece of knowledge should have a single authoritative source. Two copies of a tax-rate constant *will* drift; three copies of a validation rule *will* eventually disagree.

> ⚠ **Beware over-DRY.** Two pieces of code that look similar but change for *different* reasons are not duplication — they're coincidental similarity. Forcing them into a shared abstraction couples unrelated concerns and creates the "wrong abstraction", which is harder to fix than the duplication. Sandi Metz's rule of thumb: **prefer duplication over the wrong abstraction.**

**KISS — Keep It Simple.** If a `for` loop is clearer than a stream pipeline with six stages, write the loop. The simplest solution that solves the problem wins. Cleverness is a cost.

**YAGNI — You Aren't Gonna Need It.** Don't build features speculatively. "We might need this later" → build it later, when the requirement is clear. Speculative code has maintenance cost with zero current value, and the actual requirement when it arrives is rarely the one you guessed.

### 5.3 Composition vs inheritance — cross-link to [03](./03_oop_language.md#57-composition-vs-inheritance)

A SOLID-aware codebase **prefers composition over inheritance** because:

- Composition makes DIP natural — you inject the collaborator.
- Composition tends to keep SRP — each collaborator has its own job.
- Composition avoids LSP traps — there's no parent contract to honour.

Inheritance is appropriate only when the *is-a* relationship is genuine and Liskov Substitution holds. Otherwise: extract an interface and inject. Full discussion in [03 §5.7](./03_oop_language.md#57-composition-vs-inheritance).

### 5.4 SOLID and Spring DI: DIP in action

Spring's whole reason for existing is to make **DIP** ergonomic:

```java
// You write:
@Service public class OrderService {
    private final OrderRepository repo;                          // depends on abstraction
    public OrderService(OrderRepository repo) { this.repo = repo; }
}
@Repository public class JpaOrderRepository implements OrderRepository { /* ... */ }

// Spring's IoC container at startup:
//   1. Scans for @Service, @Repository, @Component
//   2. Sees OrderService needs an OrderRepository
//   3. Sees JpaOrderRepository implements OrderRepository
//   4. Wires them: new OrderService(new JpaOrderRepository(...))
// Your high-level code never says "new JpaOrderRepository". Pure DIP.
```

This is also why **constructor injection** is the recommended Spring style: the dependencies are explicit (you can read them off the constructor), required (compiler enforces non-null), and final (can't be reassigned). Field injection (`@Autowired` directly on a field) loses all three properties — it's why most Spring style guides discourage it.

### 5.5 Common violations to spot in code review

A short hit-list of patterns that almost always indicate a SOLID smell:

| Pattern in code | Likely SOLID violation |
|---|---|
| `if (x instanceof Foo) { ... } else if (x instanceof Bar) { ... }` in callers | LSP — subtype isn't substitutable; or OCP — switch should be polymorphic. |
| `// when X changes also remember to change Y` | SRP — two reasons to change, both in this class. |
| Empty method overrides (`@Override void foo() {}`) | ISP — the parent interface forces methods this subtype doesn't have. |
| `private final XService thing = new XServiceImpl()` | DIP — high-level code depends on a concrete impl. |
| `if/else` chain on a `type` enum that grows monthly | OCP — should be polymorphic dispatch. |
| Subclass override that throws `UnsupportedOperationException` | LSP — subtype refuses an operation the parent promised. |
| Tests that use `Mockito.mockStatic(...)` to stub a singleton | DIP — the dependency should be injected. |

---

## 6. Memory Aids

### Decision tree: "should I split this class?"

```
Does the class change for more than one reason / stakeholder?
├── Yes → split (SRP).
└── No
    ├── Does adding a new variant force editing this class?
    │   └── Yes → introduce an interface and dispatch polymorphically (OCP).
    └── Does this class have many optional methods that subclasses don't all use?
        └── Yes → split the interface (ISP).
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Single Responsibility?" | One reason to change | "One stakeholder, one axis of change. A class doing four unrelated things has four reasons to change." |
| "Open/Closed?" | Add behaviour by adding code, not editing | "Strategy + DI is the canonical OCP-friendly structure." |
| "Liskov Substitution?" | Subtypes must work everywhere the parent works | "Square-extends-Rectangle is the textbook violation." |
| "Interface Segregation?" | Many small > one fat | "Empty methods on impls are the smell." |
| "Dependency Inversion?" | High-level depends on abstraction | "What Spring DI implements. Inject interfaces, not concrete classes." |
| "Why composition over inheritance?" | Lower coupling, easier testing | "Composition supports DIP; inheritance creates fragile-base-class coupling." |
| "DRY taken too far?" | Coincidental similarity | "Two pieces that change for different reasons aren't duplication. Don't force a shared abstraction." |

### Three anchor pictures

1. **One reason to change** = one class.
2. **Inject the abstraction** — never `new` a concrete dependency in high-level code.
3. **A subtype must work where the parent works.** No exceptions. No `instanceof` switches in callers.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: SRP — Single Responsibility?
**A:** A class should have one and only one reason to change. "Reason" = stakeholder or axis of change. An OrderService that handles emails, PDFs, persistence, and audit has four reasons to change — split it.

### Q2: OCP — Open/Closed?
**A:** Software entities should be open for extension, closed for modification. Add new behaviour by adding new classes (typically Strategy implementations), not by editing existing code. Achieved via polymorphism + DI.

### Q3: LSP — Liskov Substitution?
**A:** Subtypes must be substitutable for their parents without breaking correctness. Classic violation: Square extends Rectangle — setting width also changes height, breaking Rectangle's contract.

### Q4: ISP — Interface Segregation?
**A:** Many small, focused interfaces beat one fat interface. Clients shouldn't depend on methods they don't use. Empty `@Override` methods are an ISP smell.

### Q5: DIP — Dependency Inversion?
**A:** Depend on abstractions, not concretions. High-level modules don't depend on low-level modules — both depend on an interface (typically owned by the high-level side). Spring DI is DIP in action.

### Q6: How does Spring DI implement DIP?
**A:** You declare `@Service`/`@Repository` classes that depend on interfaces via constructor parameters. Spring's `ApplicationContext` scans, finds implementations, and wires them. Your high-level code never references a concrete impl class.

### Q7: DRY — Don't Repeat Yourself?
**A:** Single source of truth for every piece of knowledge. Duplicated code → duplicated bugs. But: don't over-DRY — coincidental similarity (two things that look alike but change for different reasons) is not duplication, and forcing a shared abstraction can be worse than the duplication.

### Q8: KISS — Keep It Simple?
**A:** The simplest solution that solves the problem wins. If a for-loop is clearer than a six-stage stream pipeline, use the for-loop. Cleverness is a cost.

### Q9: YAGNI — You Aren't Gonna Need It?
**A:** Don't build features speculatively. Build for the requirement you have now; refactor when the next requirement arrives. Speculative code has maintenance cost and is usually wrong about the actual future need.

### Q10: Composition vs inheritance — when each?
**A:** Default to composition — looser coupling, supports DIP, no fragile-base-class problem. Inheritance only when "is-a" genuinely holds and Liskov Substitution is preserved. See [03 §5.7](./03_oop_language.md#57-composition-vs-inheritance).

### Q11: Show me an LSP violation other than Square/Rectangle.
**A:** A `ReadOnlyList` extending `List` and throwing on `add` — strengthens preconditions, breaks the parent's contract. Or a `LoggingDB extends DB` that drops the logging if the underlying DB returns null — weakens postconditions.

### Q12: What's a "god class"?
**A:** A class that knows about and does too many things — usually 500+ lines, multiple responsibilities. Classic SRP violation. Common in legacy code; a cue for staged refactoring.

### Q13: Why is field injection discouraged in Spring?
**A:** Hides dependencies (not visible from constructor), bypasses `final`, makes the class hard to instantiate without Spring (hurts testing). Constructor injection makes dependencies explicit, required, and final.

### Q14: Why does OCP often appear together with the Strategy pattern?
**A:** Strategy is the natural way to make a class "closed for modification" while keeping it "open for extension" — new strategies plug in via DI without touching the orchestrator.

### Q15: When is duplication preferable to abstraction?
**A:** When the two pieces only *look* similar but change for different reasons. Forcing a shared abstraction couples unrelated concerns and locks in the wrong joints. The "wrong abstraction" is harder to remove than duplication.

### Q16: SOLID priority order for practical work?
**A:** Roughly DIP > SRP > LSP > OCP > ISP. DIP and SRP carry most practical weight at scale. LSP catches concrete bugs. OCP and ISP are valuable refinements but rarer to need.

---

### Key Code Patterns

**Constructor injection (DIP)**
```java
@Service
public class OrderService {
    private final OrderRepository repo;
    private final NotificationService notifications;
    public OrderService(OrderRepository repo, NotificationService n) {
        this.repo = repo; this.notifications = n;
    }
}
```

**OCP-friendly dispatch (Strategy + DI)**
```java
@Service
public class PaymentProcessor {
    private final List<PaymentHandler> handlers;
    public PaymentProcessor(List<PaymentHandler> handlers) { this.handlers = handlers; }
    public void process(Payment p) {
        handlers.stream().filter(h -> h.supports(p.type())).findFirst().orElseThrow().process(p);
    }
}
```

**Liskov-safe sibling design**
```java
sealed interface Shape permits Rectangle, Square { double area(); }
record Rectangle(double w, double h) implements Shape { public double area() { return w * h; } }
record Square(double s) implements Shape { public double area() { return s * s; } }
```

---

## 8. Self-Test

**Easy**
- [ ] State all five SOLID principles in one sentence each.
- [ ] What does DRY mean? When does it go too far?
- [ ] What does KISS mean? YAGNI?
- [ ] Is "field injection" SOLID-friendly? Why not?

**Medium**
- [ ] Refactor a god-class `OrderService` (does email, PDF, DB, audit) into SRP-friendly collaborators.
- [ ] Write an OCP-friendly version of an `if/else` chain on a `PaymentType` enum with three current cases.
- [ ] Show how Spring DI implements DIP — with the specific class annotations involved.
- [ ] Why does `Square extends Rectangle` violate LSP — and how do you fix it?

**Hard**
- [ ] Pick a class from a codebase you've worked on and identify which SOLID principles it follows or violates.
- [ ] Argue for and against extracting an interface for a `MailSender` that has exactly one implementation.
- [ ] Why is over-DRY worse than duplication? Give a concrete example.
- [ ] Show a subtle LSP violation that doesn't involve overriding (think pre/post-conditions or thrown exceptions).
- [ ] When would you intentionally violate SRP, and why is that defensible?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **SRP** | One class, one reason to change. |
| **OCP** | Open for extension, closed for modification — add behaviour without editing existing code. |
| **LSP** | Subtypes must work where the base type works — no surprises. |
| **ISP** | Many small interfaces beat one fat one. |
| **DIP** | Depend on abstractions, not concretions. |
| **DRY** | One source of truth for every piece of knowledge. |
| **KISS** | Simpler beats cleverer. |
| **YAGNI** | Don't build for the future you guessed. |
| **God class** | A class that does too many things. SRP-violating. |
| **Fragile base class** | An inheritance hierarchy where small parent changes break subclasses. |
| **Wrong abstraction** | A shared abstraction forced over coincidentally-similar code; harder to remove than duplication. |
| **IoC / DI container** | A framework component (e.g. Spring) that wires DIP-shaped graphs at startup. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/09_solid_doubts.md) · [← Prev: 08 Design Patterns](./08_design_patterns.md) · [Next: 10 Testing →](./10_testing.md)

[↑ Back to top](#09--solid--design-principles)
