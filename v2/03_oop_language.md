# 03 — OOP & Language Fundamentals

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## 🟢 Start Here — OOP in Plain English

### What is Object-Oriented Programming?

OOP is about modelling the real world. Instead of writing one long script, you create **objects** — things that have **data** (what they know) and **behaviour** (what they can do).

```java
// A Dog object:
//   Data:     name = "Rex", breed = "Labrador", age = 3
//   Behaviour: bark(), fetch(), eat()

class Dog {
    String name;         // data (what the dog knows about itself)
    String breed;
    int age;
    
    void bark() {        // behaviour (what the dog can do)
        System.out.println(name + " says Woof!");
    }
}

Dog rex = new Dog();     // create an object (a specific dog)
rex.name = "Rex";
rex.bark();              // "Rex says Woof!"
```

### The four big ideas

**1. Encapsulation** — Hide the messy details, show only what's needed.
> Think of a TV remote. You press "volume up" — you don't need to know how the circuits work inside. The buttons are the *public interface*, the wiring is *private*.

```java
class BankAccount {
    private double balance;          // hidden — nobody can touch this directly
    
    public void deposit(double amount) {   // controlled access
        if (amount > 0) balance += amount;  // validation protects the data
    }
}
```

**2. Inheritance** — Create new things based on existing things.
> A "Sports Car" is a type of "Car". It has everything a car has (engine, wheels) plus extra stuff (turbo, spoiler).

```java
class Animal { void breathe() { } }
class Dog extends Animal { void bark() { } }  // Dog inherits breathe() and adds bark()
```

**3. Polymorphism** — Same action, different behaviour depending on the object.
> Tell any animal to "speak" — a dog barks, a cat meows, a cow moos. Same command, different result.

```java
Animal a = new Dog();    // variable type is Animal, but actual object is a Dog
a.speak();               // calls Dog's version of speak() — "Woof!"
```

**4. Abstraction** — Define *what* something does without saying *how*.
> A "payment method" can charge money. You don't care if it's a credit card, PayPal, or crypto — you just call `charge()`.

```java
interface PaymentMethod {
    void charge(double amount);   // WHAT it does (contract)
}
// HOW it does it is up to each implementation
```

### Interface vs Class — the simplest explanation

- **Class** = a blueprint for creating objects (has data + code)
- **Interface** = a contract / promise ("I can do these things") — no data, just method signatures
- A class can implement multiple interfaces but extend only one other class

### What's `final`? What's `static`?

- **`final`** = "this can't change." Final variable = can't reassign. Final class = can't extend. Final method = can't override.
- **`static`** = "belongs to the class, not any specific object." All objects share it. Like a classroom whiteboard that all students share.

> Once you understand that objects = data + behaviour, and these four concepts let you organise code cleanly — you're ready for the deeper material below.

---

## 📚 Study Material

### 1. Four Pillars of OOP

**Encapsulation** — Bundle data + methods, hide internals.
```java
public class BankAccount {
    private double balance;                       // hidden state
    public void deposit(double amount) {          // controlled access
        if (amount <= 0) throw new IllegalArgumentException("Must be positive");
        this.balance += amount;
    }
    public double getBalance() { return balance; } // read-only exposure
}
// WHY: Invariants are enforced — no one can set balance to -1000
```

**Abstraction** — Expose *what*, hide *how*.
```java
public interface PaymentGateway {
    PaymentResult charge(Money amount, Card card);  // WHAT it does
}
// Caller doesn't know if it's Stripe, PayPal, or a mock
// Implementation details are hidden behind the contract
```

**Inheritance** — Reuse + specialise via "is-a" relationship.
```java
public class SavingsAccount extends BankAccount {
    private double interestRate;
    public void applyInterest() {
        deposit(getBalance() * interestRate);  // reuses parent's deposit()
    }
}
// ⚠️ Inheritance creates tight coupling — prefer composition when "is-a" doesn't hold
```

**Polymorphism** — Same interface, different behaviour at runtime.
```java
List<BankAccount> accounts = List.of(new SavingsAccount(), new CheckingAccount());
for (BankAccount acc : accounts) {
    acc.calculateFees();  // RUNTIME dispatch — calls the correct subclass method
}
// Compile-time type: BankAccount  |  Runtime type: SavingsAccount or CheckingAccount
```

### 2. Abstract Class vs Interface (Modern Java)

```java
// INTERFACE — pure contract + default behaviour (Java 8+)
public interface Auditable {
    String getAuditId();                           // abstract — implementor must provide
    default String auditLog() {                    // default — shared implementation
        return getAuditId() + " at " + Instant.now();
    }
    static Auditable none() { return () -> "N/A"; } // static factory (Java 8)
    private String format() { return "..."; }       // private helper (Java 9)
}

// ABSTRACT CLASS — partial implementation + state
public abstract class BaseRepository<T> {
    protected final DataSource ds;                  // instance state (interfaces can't have this)
    protected BaseRepository(DataSource ds) { this.ds = ds; }
    abstract T findById(Long id);                   // subclass provides
    public void save(T entity) { /* shared impl */ }
}
```

| Feature | Interface | Abstract Class |
|---------|:---------:|:--------------:|
| Multiple inheritance | ✅ (implement many) | ❌ (extend one) |
| Instance fields | ❌ (only `static final`) | ✅ |
| Constructors | ❌ | ✅ |
| Default methods | ✅ (Java 8+) | ✅ |
| Access modifiers on methods | `public` only (abstract) | Any |
| When to use | Define a capability contract | Share state + partial implementation |

💡 **Modern rule:** Start with interfaces. Use abstract class only when you need shared mutable state or constructor logic.

### 3. Method Dispatch — Overloading vs Overriding

```java
class Printer {
    // OVERLOADING — same name, different parameters
    // Resolved at COMPILE TIME (static dispatch) based on declared parameter types
    void print(String s)  { System.out.println("String: " + s); }
    void print(Integer i) { System.out.println("Integer: " + i); }
    void print(Object o)  { System.out.println("Object: " + o); }
}

Object obj = "hello";
new Printer().print(obj);  // prints "Object: hello" — compile-time type is Object!

class ColorPrinter extends Printer {
    // OVERRIDING — same signature as parent
    // Resolved at RUNTIME (dynamic dispatch / virtual method table)
    @Override
    void print(String s) { System.out.println("Color: " + s); }
}

Printer p = new ColorPrinter();
p.print("test");  // prints "Color: test" — runtime type is ColorPrinter
```

⚠️ **Trap:** Overloading is resolved by compile-time type of arguments. Overriding is resolved by runtime type of the object. They follow different rules.

### 4. Generics — Type Erasure & Bounded Wildcards

**Type erasure:** Generics exist only at compile time. At runtime, `List<String>` is just `List`.

```java
// At compile time: type-safe
List<String> names = new ArrayList<>();
names.add("Alice");
String s = names.get(0);          // no cast needed — compiler verifies

// At runtime (after erasure):
List names = new ArrayList();      // raw type
names.add("Alice");
String s = (String) names.get(0); // compiler inserted this cast

// CONSEQUENCES of erasure:
// ❌ new T()           — T doesn't exist at runtime
// ❌ instanceof T      — T doesn't exist at runtime
// ❌ new T[10]         — can't create generic array
// ❌ catch (T e)       — can't catch type parameter
// ❌ T.class           — no class literal for type param
```

**PECS — Producer Extends, Consumer Super:**
```java
// PRODUCER (you READ from it) → use ? extends T
public double sum(List<? extends Number> nums) {
    double total = 0;
    for (Number n : nums) total += n.doubleValue();  // READ is safe — everything is a Number
    // nums.add(1);  ❌ — compiler doesn't know if it's List<Integer> or List<Double>
    return total;
}
sum(List.of(1, 2, 3));         // List<Integer> — OK
sum(List.of(1.0, 2.0));        // List<Double> — OK

// CONSUMER (you WRITE to it) → use ? super T
public void addNumbers(List<? super Integer> list) {
    list.add(1);                // WRITE is safe — list accepts Integer or any supertype
    list.add(2);
    // Integer x = list.get(0); ❌ — might be List<Number> or List<Object>
}
addNumbers(new ArrayList<Number>());    // OK
addNumbers(new ArrayList<Object>());    // OK
```

### 5. `final`, `static`, `this`, `super`

```java
// FINAL — three contexts
final int x = 10;              // variable: can't reassign (reference is fixed)
final List<String> list = new ArrayList<>();
list.add("OK");                // object contents CAN change — only reference is final

class Parent {
    final void validate() {}   // method: can't be overridden
}

final class ImmutablePoint {   // class: can't be subclassed (String, Integer, records)
    final int x, y;
}

// STATIC — class-level, shared across all instances
class Counter {
    static int count = 0;      // ONE copy shared by all instances
    static void reset() { count = 0; }  // no `this` — can't access instance fields
}
// ⚠️ Static mutable state is dangerous: race conditions, test pollution, tight coupling

// THIS — reference to current instance
class Node {
    int value;
    Node(int value) { this.value = value; }  // disambiguate param vs field
    Node next() { return this; }             // return self for fluent API
}

// SUPER — reference to parent class
class Child extends Parent {
    Child() { super(); }                     // call parent constructor (must be first line)
    @Override void validate() { super.validate(); }  // call parent's version
}
```

### 6. Records (Java 16+)

```java
// Immutable data carrier — compiler generates everything
public record Money(BigDecimal amount, Currency currency) {
    // Auto-generated: private final fields, canonical constructor,
    // equals(), hashCode(), toString(), accessor methods (amount(), currency())

    // Compact constructor — validation without repeating assignments
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount must be non-negative");
        }
        amount = amount.setScale(2, RoundingMode.HALF_UP);  // normalise
        // assignments to this.amount and this.currency happen implicitly AFTER
    }

    // Custom methods are fine
    public Money add(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException();
        return new Money(amount.add(other.amount), currency);
    }
}

// ⚠️ Records are shallowly immutable — if a component is mutable (e.g., List),
//    you must make defensive copies:
public record Team(String name, List<String> members) {
    public Team {
        members = List.copyOf(members);  // defensive copy — caller can't mutate
    }
}
```

### 7. Sealed Classes (Java 17+)

```java
// Restrict which classes can extend — exhaustive type hierarchies
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}
public record Rectangle(double w, double h) implements Shape {
    public double area() { return w * h; }
}
public final class Triangle implements Shape { /* ... */ }

// Exhaustive switch (Java 21) — compiler verifies all cases covered
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.w() * r.h();
    case Triangle t  -> t.area();
    // no default needed — sealed interface guarantees exhaustiveness
};
```

### 8. Composition vs Inheritance

```java
// INHERITANCE — rigid, fragile
class EmailNotifier extends Logger {     // "is-a" Logger? Doesn't make sense.
    void sendAlert(String msg) {
        log(msg);                        // inherits log() — but that's misuse of inheritance
        emailService.send(msg);
    }
}

// COMPOSITION — flexible, swappable
class AlertService {
    private final Logger logger;          // "has-a" Logger
    private final EmailService email;     // "has-a" EmailService
    
    AlertService(Logger logger, EmailService email) {  // inject — easy to test with mocks
        this.logger = logger;
        this.email = email;
    }
    void sendAlert(String msg) {
        logger.log(msg);
        email.send(msg);
    }
}
// Composition is preferred because:
// 1. No tight coupling to parent implementation
// 2. Can swap components at runtime
// 3. Easy to test (inject mocks)
// 4. No fragile base class problem
```

**Rule of thumb:** Use inheritance only when Liskov Substitution genuinely holds (every subclass instance can substitute for the parent without breaking correctness).

### 9. Pattern Matching (Java 16+)

```java
// OLD — verbose instanceof + cast
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// NEW — pattern matching eliminates the cast
if (obj instanceof String s) {
    System.out.println(s.length());      // s is already typed
}

// Guarded pattern
if (obj instanceof String s && s.length() > 5) { /* ... */ }

// Switch pattern matching (Java 21)
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "positive int: " + i;
        case Integer i            -> "non-positive int: " + i;
        case String s             -> "string of length " + s.length();
        case null                 -> "null";
        default                   -> "other: " + obj;
    };
}
```

---

## Rapid-Fire Q&A

### Q1: Interface vs abstract class — when each?
**A:** Interface = pure contract. Since Java 8: default methods (shared behaviour) and static methods — but no instance state. Abstract class = partial implementation + instance state. Modern preference: program to interfaces. Use abstract class only when you need shared mutable state or a template method pattern.

### Q2: What changed about interfaces in Java 8?
**A:** Added `default` methods (implementation in interface — avoids breaking existing implementations). Added `static` methods. Java 9 added `private` methods in interfaces. This moved Java closer to traits / mix-in patterns.

### Q3: Method overloading vs overriding?
**A:** Overloading = same name, different parameters. Resolved at **compile time** (static dispatch). Overriding = subclass replaces superclass method (same signature). Resolved at **runtime** (dynamic dispatch / virtual method lookup). `@Override` annotation catches typos at compile time.

### Q4: `final` keyword — three contexts?
**A:** `final` variable: can't reassign (reference is fixed, but object contents can still change). `final` method: can't be overridden by subclasses. `final` class: can't be subclassed (e.g., `String`, `Integer`, records).

### Q5: `static` — what does it mean? Why is static mutable state dangerous?
**A:** Class-level, not instance-level. Shared across all instances. Static mutable state is dangerous because: (1) all threads share it — race conditions, (2) test pollution — one test's state leaks to another, (3) tight coupling — hard to mock.

### Q6: Checked vs unchecked exceptions?
**A:** Checked (`IOException`, `SQLException`): must be declared or caught. Use for recoverable conditions. Unchecked (`RuntimeException` subtypes — `NullPointerException`, `IllegalArgumentException`): compiler doesn't enforce. Use for programming errors. Modern bias: lean on unchecked — checked exceptions don't compose well with lambdas/streams.

### Q7: Try-with-resources — what interface does it require?
**A:** `AutoCloseable` (or the older `Closeable`). Compiler generates `finally` block that calls `close()`. If both the try block and `close()` throw, the close exception is **suppressed** (attached to the primary exception via `getSuppressed()`).

### Q8: Generics — what is type erasure? Why can't you do `new T()`?
**A:** Generics are compile-time only. At runtime, `List<String>` is just `List` (raw type). `T` doesn't exist at runtime — the compiler erases it. So `new T()` is impossible (no type info). Also can't do `instanceof T`, `new T[]`, or catch `T`.

### Q9: PECS — what does it stand for?
**A:** Producer Extends, Consumer Super. `? extends T` — read-only (produce T-or-subtypes out). `? super T` — write-only (consume T-or-subtypes in). `Collection<? extends Number>` lets you read `Number` but not write (compiler can't know if it's `List<Integer>` or `List<Double>`).

### Q10: `Optional` — when appropriate? When wrong?
**A:** Good: return type for "might not exist" lookups (`findById`). Bad: don't use for fields, method parameters, or collections (use empty collection instead). Prefer `orElse`, `orElseGet`, `map`, `ifPresent` over `get()` + `isPresent()`.

### Q11: `equals` vs `==`?
**A:** `==` compares references (same object in memory). `equals()` compares logical equality (value). `String` pool: `"hello" == "hello"` is true (same interned instance), but `new String("hello") == "hello"` is false. Always use `equals()` for objects.

### Q12: What's an enum? When use it?
**A:** Type-safe constant set. Can have methods, fields, constructors. Use instead of `int` constants. Can implement interfaces. `EnumSet` and `EnumMap` are optimised implementations. Can be used in `switch` statements. Implicitly `final` and serialisation-safe singletons.

### Q13: What is `record` (Java 16+)?
**A:** Immutable data carrier. Auto-generates: `private final` fields, constructor, `equals`, `hashCode`, `toString`, getters (`name()` not `getName()`). No setters. Can have compact constructors for validation. Still need defensive copies for mutable field types.

### Q14: What are sealed classes (Java 17+)?
**A:** Restrict which classes can extend/implement. `sealed class Shape permits Circle, Rectangle`. Forces exhaustive pattern matching in `switch`. Between `final` (no subclasses) and open (any subclass).

### Q15: What's the diamond problem and how does Java handle it?
**A:** Two interfaces provide conflicting `default` methods. Java forces the implementing class to override and resolve explicitly. Abstract class can only have single inheritance — no diamond problem there.

### Q16: Composition vs Inheritance — when each?
**A:** Modern preference: composition ("has-a"). Inheritance ("is-a") creates rigid hierarchies, breaks encapsulation (subclass depends on superclass implementation). Composition is more flexible — swap behaviour at runtime. Use inheritance only when Liskov Substitution genuinely holds.

### Q17: What's `instanceof` and pattern matching (Java 16+)?
**A:** `instanceof` checks type at runtime. Pattern matching eliminates the cast: `if (obj instanceof String s)` — `s` is already typed. Combines with sealed classes for exhaustive `switch` (Java 21).

### Q18: Immutability — how to create an immutable class?
**A:** `final` class, `private final` fields, no setters, values via constructor only, defensive copy mutable fields on input AND output. Records give you most of this for free (but still need defensive copies for mutable types like `List`).

---

## Can you answer these cold?

- [ ] Interface vs abstract class — with Java 8 default methods context
- [ ] Overloading (compile time) vs overriding (runtime) — explain dispatch
- [ ] `final` in three contexts — variable, method, class
- [ ] Type erasure — what's lost at runtime, what can't you do
- [ ] PECS — explain with a method signature example
- [ ] Checked vs unchecked — when to use each, why streams prefer unchecked
- [ ] Composition vs inheritance — give a real refactoring example
- [ ] `record` vs regular class — what you get for free, what you don't

[← Back to Index](./00_INDEX.md)
