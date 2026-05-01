# 09 — SOLID & Design Principles

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

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
