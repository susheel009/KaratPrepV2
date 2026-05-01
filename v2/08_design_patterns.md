# 08 — Design Patterns

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

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
