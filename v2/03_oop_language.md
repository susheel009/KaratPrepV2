# 03 — OOP & Language Features

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/03_oop_language_doubts.md) · [← Prev: 02 Collections](./02_collections.md) · [Next: 04 Exceptions →](./04_exceptions.md)
>
> **Priority:** 🔴 Critical · **Related topics:** [06 Java 8+ (functional interfaces, records)](./06_java8_plus.md) · [09 SOLID](./09_solid.md) · [04 Exceptions](./04_exceptions.md) · [02 Collections (equals/hashCode, generics)](./02_collections.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — why we group code into objects
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — overload (compile time) vs override (runtime)
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — minimal hierarchy
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Encapsulation](#41-encapsulation--hiding-the-wires)
   - [4.2 Inheritance](#42-inheritance--is-a-relationships)
   - [4.3 Polymorphism](#43-polymorphism--same-call-different-behaviour)
   - [4.4 Abstraction (interfaces)](#44-abstraction--interfaces-as-contracts)
   - [4.5 Overloading vs overriding](#45-overloading-vs-overriding--two-different-kinds-of-dispatch)
   - [4.6 final and static](#46-final-and-static)
   - [4.7 Access modifiers](#47-access-modifiers--public-protected-package-private-private)
   - [4.8 Inner / nested / anonymous classes](#48-inner--nested--anonymous-classes)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 Generics + type erasure](#51-generics--type-erasure)
   - [5.2 PECS (bounded wildcards)](#52-pecs--bounded-wildcards)
   - [5.3 Records (Java 16+)](#53-records-java-16)
   - [5.4 Sealed classes (Java 17+)](#54-sealed-classes-java-17)
   - [5.5 Pattern matching](#55-pattern-matching-java-16-instanceof-java-21-switch)
   - [5.6 Default / static / private interface methods](#56-default--static--private-interface-methods)
   - [5.7 Composition vs inheritance](#57-composition-vs-inheritance)
   - [5.8 Immutability](#58-immutability--how-to-build-an-immutable-class)
   - [5.9 Diamond resolution](#59-the-diamond-problem-and-how-java-resolves-it)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

Imagine you're running a coffee-shop chain. **Head office** writes one document — the **brand contract**: every shop must serve filtered drip, espresso, and tea; close at 9pm; recycle cups. The contract doesn't say *how* — that's up to each shop. Cleveland uses a Bunn drip machine; Seattle has a manual pour-over bar. Both honour the contract.

The chain expands. New shop types open: drive-thru, sit-down café, kiosk-in-a-bookstore. To save head office from rewriting the whole contract for each, they introduce a **partly-finished blueprint** — a document with the shared bones of any shop (point-of-sale procedure, opening checklist, cleaning rota), with blanks for shop-specific fields ("seating capacity: ___", "drive-thru lanes: ___"). Each shop type fills in the blanks differently and adds its own quirks.

The actual **buildings** — 1,200 of them — are *instances*. Each one is a real object: the espresso machine, the cash register, the people, the daily revenue. Every building obeys the same brand contract, fills in the same blueprint, and yet has its own state.

That's object-oriented programming.

- The **brand contract** = an **interface**: a list of capabilities a shop must provide. No state, no implementation.
- The **partly-finished blueprint** = an **abstract class**: shared bones plus blanks for subclasses to fill. Has fields, has constructor logic, and has methods — some implemented, some abstract.
- A **specific shop type** = a **concrete class**: every blank filled in.
- A **specific building** = an **object** (a.k.a. **instance**): real memory, real state, real behaviour.

> Once you see classes as the templates and objects as the buildings, four words suddenly mean less than you feared: **encapsulation** (the espresso machine doesn't show its wiring), **inheritance** (drive-thru *is a* shop), **polymorphism** (everyone honours the brand contract differently), **abstraction** (head office never sees the wiring either, only the contract).

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

Java has **two superficially similar but fundamentally different** mechanisms for "the same name doing different things":

**Overloading** — same method name, different parameter list, **chosen at compile time** by the compiler based on the argument types it sees in the source.

**Overriding** — subclass replaces a parent's method with the same signature, **chosen at runtime** by the JVM based on the actual object's type.

Walk through both side-by-side:

```
class Printer:
    print(String s)  → "String: …"
    print(Object o)  → "Object: …"

class ColorPrinter extends Printer:
    print(String s)  → "Color: …"     // override

Object obj = "hello"                  // declared type: Object, runtime type: String
Printer p   = new Printer()
p.print(obj)                          // OVERLOAD chosen by compile-time type "Object"
                                      //   → "Object: hello"   ← surprises beginners

Printer cp  = new ColorPrinter()      // declared type: Printer, runtime type: ColorPrinter
cp.print("hi")                        // OVERRIDE chosen by runtime type "ColorPrinter"
                                      //   → "Color: hi"
```

**Two rules in one walkthrough:**

1. **Overloading is compile-time.** The compiler looks at the *declared* type of every argument and picks the matching method. The runtime never gets to vote. So `p.print(obj)` calls `print(Object)` even though `obj` actually points to a `String`.
2. **Overriding is runtime.** The JVM looks up the method via the object's *real* class via the virtual method table (vtable). So `cp.print("hi")` calls `ColorPrinter`'s version even though `cp`'s declared type is `Printer`.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
class Animal {
    final String name;
    Animal(String name) { this.name = name; }       // constructor — sets immutable field

    void speak() { System.out.println(name + " makes a sound"); }  // base behaviour
}

class Dog extends Animal {
    Dog(String name) { super(name); }                // call parent constructor
    @Override void speak() { System.out.println(name + " barks"); }  // OVERRIDE
}

class Cat extends Animal {
    Cat(String name) { super(name); }
    @Override void speak() { System.out.println(name + " meows"); }
}

public class Demo {
    public static void main(String[] args) {
        Animal[] zoo = { new Dog("Rex"), new Cat("Lily"), new Animal("Goldie") };
        for (Animal a : zoo) {                       // declared type Animal,
            a.speak();                               // but JVM dispatches by RUNTIME type
        }                                            // → "Rex barks" / "Lily meows" / "Goldie makes a sound"
    }
}
```

What just happened: one loop, three different behaviours, no `if/else`. The JVM's virtual-method dispatch picks the right `speak()` based on each element's actual runtime class. That's polymorphism in 12 lines.

---

## 4. Build Up — Practical Patterns

### 4.1 Encapsulation — hiding the wires

Bundle the data and the methods that operate on it together; hide internals so callers can't break invariants.

```java
public class BankAccount {
    private double balance;                                 // hidden — nobody touches directly
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("must be positive");
        this.balance += amount;
    }
    public double getBalance() { return balance; }          // controlled read
}
```

**Why:** invariants are enforced. No one can drop `balance` to `-1000` from the outside, ever. Changing the storage (e.g. switching to `BigDecimal`) is a one-line change because no code depends on the field.

### 4.2 Inheritance — "is-a" relationships

A subclass inherits everything its parent has and may add or override.

```java
public class SavingsAccount extends BankAccount {
    private final double rate;
    public SavingsAccount(double rate) { this.rate = rate; }
    public void applyInterest() {
        deposit(getBalance() * rate);                       // reuses parent's deposit logic
    }
}
```

> ⚠ **Inheritance is a heavy commitment.** The subclass becomes coupled to the parent's *implementation*, not just its public contract. A small change in the parent can silently break the subclass. Modern guidance: use inheritance only when **Liskov Substitution** genuinely holds — every `Subclass` instance can be substituted for the parent without breaking correctness — and prefer **composition** (§5.7) otherwise.

### 4.3 Polymorphism — same call, different behaviour

```java
List<BankAccount> accounts = List.of(new SavingsAccount(0.04), new CheckingAccount());
for (BankAccount acc : accounts) {
    acc.calculateFees();                          // RUNTIME dispatch — each subclass's method runs
}
```

The compiler sees `BankAccount`. The JVM looks up the *real* class at each call and picks its method. This is what makes large polymorphic systems possible — strategies, plugins, GUI listeners.

### 4.4 Abstraction — interfaces as contracts

An **interface** says *what* without committing to *how*.

```java
public interface PaymentGateway {
    PaymentResult charge(Money amount, Card card);
}

public class StripeGateway implements PaymentGateway { /* HTTP call to Stripe */ }
public class TestGateway   implements PaymentGateway { /* return canned result for tests */ }
```

The caller sees only `PaymentGateway`. Swap implementations at construction time (DI) without touching the caller.

| | Interface | Abstract class |
|---|:---:|:---:|
| Multiple inheritance | ✅ implement many | ❌ extend one |
| Instance fields | ❌ (only `static final` constants) | ✅ |
| Constructors | ❌ | ✅ |
| Default methods | ✅ (Java 8+) | ✅ |
| Method access | `public` only | any |
| When to use | A capability contract | Shared state + partial impl + template-method |

Modern rule: **start with interfaces**. Use an abstract class only when you genuinely need shared mutable state or constructor logic.

### 4.5 Overloading vs overriding — two different kinds of dispatch

```java
class Printer {
    // OVERLOADING — same name, different parameter types. Compile-time dispatch.
    void print(String s)  { System.out.println("String: " + s); }
    void print(Integer i) { System.out.println("Integer: " + i); }
    void print(Object o)  { System.out.println("Object: " + o); }
}

Object obj = "hi";
new Printer().print(obj);   // "Object: hi"  — chosen by DECLARED type Object

class ColorPrinter extends Printer {
    // OVERRIDING — same signature as parent. Runtime dispatch.
    @Override void print(String s) { System.out.println("Color: " + s); }
}

Printer p = new ColorPrinter();
p.print("hi");              // "Color: hi"   — chosen by RUNTIME type ColorPrinter
```

| | Overloading | Overriding |
|---|---|---|
| Where | Same class (or inherited) | Subclass replaces parent's method |
| Match | Different parameter list | Same signature |
| Decision | Compile time (static dispatch) | Runtime (vtable / dynamic dispatch) |
| Annotation | none | `@Override` (catches typos) |

> ⚠ Always use `@Override`. Without it, a typo (`equlas` instead of `equals`) silently creates an overload, and your "override" never runs.

### 4.6 `final` and `static`

```java
// FINAL — three contexts:
final int x = 10;                   // variable: cannot reassign
final List<String> list = new ArrayList<>();
list.add("OK");                     // contents CAN change — only the reference is final

class Parent { final void check() {} }      // method: cannot be overridden
final class Money {  /* …  */ }              // class: cannot be subclassed (String, Integer, records)

// STATIC — class-level, shared across all instances
class Counter {
    static int count = 0;            // ONE copy shared by ALL Counter instances
    static void reset() { count = 0; }       // no `this` — can't access instance fields
    static final int MAX = 100;       // common idiom for compile-time constants
}
```

> ⚠ **Static mutable state is dangerous.** All threads share it (race conditions), test pollution leaks across test methods, and it makes mocking hard. `static final` constants are fine; `static`-mutable fields almost never are.

### 4.7 Access modifiers — public, protected, package-private, private

| Modifier | Same class | Same package | Subclass (any package) | Anywhere |
|----------|:---:|:---:|:---:|:---:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| (none — package-private) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

**Default to `private`.** Widen only when there's a need. Most class internals should be private; the public API surface should be deliberately small.

### 4.8 Inner / nested / anonymous classes

```java
public class Outer {
    private int x = 1;

    // 1. STATIC NESTED CLASS — no implicit reference to the outer instance.
    //    Use for helper types tightly coupled to the outer (e.g. a Builder).
    static class Builder { /* … */ }

    // 2. INNER CLASS (non-static) — implicit reference to the enclosing Outer instance.
    //    Use rarely; carries the outer reference even when you don't need it.
    class Inner {
        int read() { return x; }    // accesses outer's private x
    }

    // 3. LOCAL CLASS — declared inside a method.
    void foo() {
        class Local { /* visible only inside foo() */ }
    }

    // 4. ANONYMOUS CLASS — one-shot subclass / impl, no name.
    Runnable r = new Runnable() {
        @Override public void run() { System.out.println(x); }
    };
}
```

> 💡 Since Java 8, lambdas (`Runnable r = () -> {...}`) replace most anonymous-class use cases. Reach for an anonymous class only when you need extra fields or multiple methods.

---

## 5. Going Deep — Interview-Level Material

### 5.1 Generics + type erasure

```java
// Compile time: type-safe
List<String> names = new ArrayList<>();
names.add("Alice");
String s = names.get(0);          // no cast — compiler verifies

// Runtime: erased to raw List
// Conceptually:
List names = new ArrayList();
names.add("Alice");
String s = (String) names.get(0); // compiler-inserted cast

// Consequences of erasure:
T x       = new T();              // ❌ T doesn't exist at runtime
boolean b = obj instanceof T;     // ❌ same
T[] arr   = new T[10];            // ❌ same
catch (T e) { /* … */ }           // ❌ same
Class<T> c = T.class;             // ❌ no class literal for type parameter

// Two raw-type traps:
List<String> ls = new ArrayList<>();
List raw = ls;                    // unchecked — compiles with warning
raw.add(42);                      // HEAP POLLUTION — Integer in a List<String>
String boom = ls.get(0);          // ClassCastException at the cast site (not the add)
```

`@SafeVarargs` exists because varargs of generic types are inherently unchecked — the JVM allocates an `Object[]` and casts. Apply it only when you don't write to the array.

### 5.2 PECS — bounded wildcards

> **Producer Extends, Consumer Super.**

```java
// PRODUCER — you READ from it. Use ? extends T.
public double sum(List<? extends Number> nums) {
    double total = 0;
    for (Number n : nums) total += n.doubleValue();   // safe to read as Number
    // nums.add(1);  ❌ — unknown if List<Integer> or List<Double>; can't safely add
    return total;
}
sum(List.of(1, 2, 3));            // List<Integer>
sum(List.of(1.0, 2.0));           // List<Double>

// CONSUMER — you WRITE to it. Use ? super T.
public void addNumbers(List<? super Integer> sink) {
    sink.add(1); sink.add(2);     // safe to add Integer (or any subtype)
    // Integer x = sink.get(0);  ❌ — could be List<Number> or List<Object>
}
addNumbers(new ArrayList<Number>());
addNumbers(new ArrayList<Object>());
```

Textbook PECS: `Collections.copy(List<? super T> dest, List<? extends T> src)` — dest is the consumer (super), src is the producer (extends).

### 5.3 Records (Java 16+)

A **record** is an immutable data carrier where the compiler generates the boilerplate.

```java
public record Money(BigDecimal amount, Currency currency) {
    // Auto-generated: private final fields, canonical constructor,
    // equals(), hashCode(), toString(), accessors (amount(), currency()).

    // COMPACT CONSTRUCTOR — validation without retyping the assignments
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("non-negative");
        amount = amount.setScale(2, RoundingMode.HALF_UP);   // normalise — implicit assignment after
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException();
        return new Money(amount.add(other.amount), currency);
    }
}

// ⚠ Records are SHALLOWLY immutable. If a component is mutable, defensive-copy in the constructor:
public record Team(String name, List<String> members) {
    public Team { members = List.copyOf(members); }   // caller's list can't mutate ours
}
```

### 5.4 Sealed classes (Java 17+)

Restrict who can extend or implement — making exhaustive type hierarchies possible.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}
public record Circle(double radius)        implements Shape { public double area() { return Math.PI * radius * radius; } }
public record Rectangle(double w, double h) implements Shape { public double area() { return w * h; } }
public final class Triangle               implements Shape { public double area() { /* … */ return 0; } }

// Java 21+ exhaustive switch — compiler verifies all cases covered, no default needed
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.w() * r.h();
    case Triangle t  -> t.area();
};
```

A permitted subclass must be one of: `final`, `sealed` (with its own permits), or `non-sealed` (open further).

### 5.5 Pattern matching (Java 16+ instanceof; Java 21 switch)

```java
// Old: verbose instanceof + cast
if (obj instanceof String) {
    String s = (String) obj;
    use(s);
}

// Java 16+: pattern matching for instanceof — cast is gone
if (obj instanceof String s) use(s);

// Guarded
if (obj instanceof String s && s.length() > 5) use(s);

// Java 21: pattern matching for switch — combined with sealed for exhaustiveness
String describe(Object o) {
    return switch (o) {
        case Integer i when i > 0 -> "positive int: " + i;
        case Integer i            -> "non-positive int: " + i;
        case String s             -> "string of length " + s.length();
        case null                 -> "null";
        default                   -> "other: " + o;
    };
}
```

### 5.6 Default / static / private interface methods

| Method type on interface | Since | Purpose |
|--|:--:|--|
| Abstract (no body) | always | Capability contract |
| `default` (with body) | Java 8 | Add behaviour without breaking existing implementors |
| `static` | Java 8 | Factory methods, helpers (`Comparator.comparing`) |
| `private` | Java 9 | Share code between defaults/statics in the same interface |

```java
public interface Auditable {
    String getAuditId();                                       // abstract
    default String auditLog() { return getAuditId() + " at " + Instant.now(); }
    static Auditable none() { return () -> "N/A"; }
    private String prefix() { return "[audit] "; }              // helper for defaults — Java 9+
}
```

### 5.7 Composition vs inheritance

**Inheritance** ties the subclass to the parent's *implementation*. Every parent change is a potential subclass break (the "fragile base class" problem).

```java
// Inheritance — rigid
class EmailNotifier extends Logger { /* "is-a" Logger? Probably not. */ }

// Composition — flexible, easy to test, easy to swap
class AlertService {
    private final Logger logger;
    private final EmailService email;
    AlertService(Logger logger, EmailService email) {           // constructor injection
        this.logger = logger; this.email = email;
    }
    void sendAlert(String msg) { logger.log(msg); email.send(msg); }
}
```

> 💡 Modern guidance: **prefer composition**. Use inheritance only when the "is-a" relationship is genuine and Liskov Substitution holds.

### 5.8 Immutability — how to build an immutable class

1. `final class` (no subclasses can break invariants)
2. All fields `private final`
3. No setters — values come in via constructor only
4. Defensive-copy mutable fields on the way in *and* out
5. Don't expose internal state through methods (return copies or unmodifiable views)

```java
public final class Period {
    private final LocalDate start, end;
    private final List<Holiday> holidays;
    public Period(LocalDate start, LocalDate end, List<Holiday> holidays) {
        if (end.isBefore(start)) throw new IllegalArgumentException();
        this.start = start; this.end = end;
        this.holidays = List.copyOf(holidays);                  // defensive in
    }
    public List<Holiday> holidays() { return holidays; }        // already immutable — safe
}
```

Records give you most of this for free — except step 4 (you must defensive-copy mutable components in a compact constructor).

### 5.9 The diamond problem and how Java resolves it

If two interfaces provide a `default` method with the same signature, the implementing class **must** override and resolve explicitly:

```java
interface A { default String hello() { return "A"; } }
interface B { default String hello() { return "B"; } }

class C implements A, B {
    @Override public String hello() { return A.super.hello() + " " + B.super.hello(); }
}
```

For classes, Java's single-inheritance rule means there's no diamond: a class extends exactly one other class.

---

## 6. Memory Aids

### Decision tree: interface vs abstract class

```
Do you need to share STATE (mutable or constructor-set fields) between implementors?
├── Yes → abstract class.
└── No
    ├── Need a TEMPLATE METHOD with shared core logic + abstract steps?
    │   ├── Yes → abstract class (or, since Java 8, a default method may suffice).
    │   └── No → INTERFACE.
    └── Multiple inheritance of behaviour expected?
        └── Yes → INTERFACE (you can implement many).
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Overloading vs overriding?" | Compile time vs runtime | "Overload picked by declared types of args. Override picked by the object's runtime class." |
| "Why `@Override`?" | Catches typos at compile time | Without it, a misspelled method becomes a brand-new overload that never runs. |
| "Generics — `new T()`?" | Erasure — T doesn't exist at runtime | Pass `Class<T>` and call `clazz.getDeclaredConstructor().newInstance()`. |
| "PECS?" | Producer extends, Consumer super | "Read from a producer with `? extends T`; write to a consumer with `? super T`." |
| "Composition vs inheritance?" | Fragile base class | "Inheritance couples the subclass to the parent's implementation. Composition lets me swap and test." |
| "How do I make X immutable?" | final class, final fields, defensive copies | "And consider a record." |
| "What's a sealed class?" | Closed hierarchy | "Restricts subclasses to a permitted list. Enables exhaustive switch." |

### Three anchor problems

1. **Overload is compile-time, override is runtime.** This single distinction is in 80% of OOP interview gotchas.
2. **Generics are erased at runtime.** That's why `new T()`, `instanceof T`, `T[]`, and `T.class` all fail.
3. **`equals` and `hashCode` must move together.** Override one → override the other.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: Interface vs abstract class — when each?
**A:** Interface = pure contract; since Java 8 also default + static methods, but no instance state. Abstract class = partial implementation + state + constructors. Default to interfaces; pick abstract class when shared mutable state or constructor logic is needed.

### Q2: What changed about interfaces in Java 8?
**A:** `default` methods (so adding a new method to an interface doesn't break existing implementors) and `static` methods. Java 9 added `private` methods for sharing code among defaults.

### Q3: Method overloading vs overriding?
**A:** Overloading: same name, different parameters; resolved at compile time. Overriding: subclass replaces parent's method with same signature; resolved at runtime via the vtable. `@Override` catches typos.

### Q4: `final` keyword — three contexts?
**A:** Variable: can't reassign (the reference; contents may still mutate). Method: can't be overridden. Class: can't be subclassed (`String`, `Integer`, records).

### Q5: `static` — what does it mean? Why is static mutable state dangerous?
**A:** Class-level, not instance-level — one copy across all instances. Static mutable state risks: race conditions across threads, test pollution, hard to mock. Static `final` constants are fine.

### Q6: Generics — what is type erasure? Why can't you do `new T()`?
**A:** Generics are compile-time only. At runtime, `List<String>` is `List`. Type parameter `T` doesn't exist at runtime, so `new T()`, `instanceof T`, `new T[]`, and catching `T` are all impossible.

### Q7: PECS — what does it stand for?
**A:** Producer Extends, Consumer Super. `? extends T` lets you read T-or-subtypes (producer). `? super T` lets you write T-or-subtypes (consumer). Textbook example: `Collections.copy(List<? super T> dest, List<? extends T> src)`.

### Q8: `equals` vs `==`?
**A:** `==` compares references (same object in memory). `equals()` compares logical equality defined by the class. For `String`, `==` may be true on interned literals but false on `new String(...)`; always use `equals` for value comparison.

### Q9: What's the diamond problem and how does Java handle it?
**A:** Two interfaces providing conflicting `default` methods. The implementing class must override and resolve explicitly (e.g. with `Interface.super.method()`). Classes have single inheritance — no diamond.

### Q10: Composition vs Inheritance — when each?
**A:** Inheritance only when "is-a" genuinely holds and Liskov Substitution is preserved. Otherwise composition: more flexible, easier to test, no fragile-base-class coupling.

### Q11: What's a `record` (Java 16+)?
**A:** Immutable data carrier. Compiler generates: private final fields, canonical constructor, `equals`, `hashCode`, `toString`, accessors (`name()` not `getName()`). Compact constructors enable validation. Shallowly immutable — defensive-copy mutable components.

### Q12: What are sealed classes (Java 17+)?
**A:** Restrict subclasses to a permitted list (`sealed class Shape permits Circle, Rectangle`). Permitted subclasses must be `final`, `sealed`, or `non-sealed`. Enables exhaustive `switch` (Java 21).

### Q13: What's `instanceof` pattern matching (Java 16+)?
**A:** Combines type check with binding: `if (obj instanceof String s) { use(s); }` — `s` is the typed reference. Can be guarded with `&&`. Java 21 extends this to `switch` for exhaustive type dispatching.

### Q14: How do you create an immutable class?
**A:** `final` class, `private final` fields, no setters, all values via constructor, defensive-copy mutable inputs and outputs. Records cover most of this; you still need defensive copies for mutable components.

### Q15: What's an enum?
**A:** Type-safe constant set. Implicitly `final` extends `java.lang.Enum`. Can have fields, constructors, methods, implement interfaces. `EnumSet`/`EnumMap` are bit-vector / array backed and very fast. Single instances guaranteed (good for singletons).

### Q16: Inner classes — what are the four kinds?
**A:** Static nested (no outer reference), inner (implicit outer reference), local (inside a method), anonymous (one-shot, no name). Lambdas have replaced most anonymous-class use cases.

### Q17: Why does an inner (non-static) class capture the outer instance?
**A:** It implicitly stores a reference to the enclosing `Outer.this` so its methods can access outer's fields. Consequence: holding an inner instance keeps the outer alive — common source of memory leaks. Use `static` nested when you don't need outer access.

### Q18: What's the difference between `extends` and `implements`?
**A:** `extends` (class → class) inherits fields and methods, single inheritance. `extends` (interface → interface) extends the contract. `implements` (class → interface) commits the class to providing the interface's methods; multiple `implements` allowed.

### Q19: Checked vs unchecked exceptions in method signatures? (preview of [04 Exceptions](./04_exceptions.md))
**A:** Checked: must be declared in `throws` or caught (compiler enforced). Unchecked (`RuntimeException` and subtypes): no compiler enforcement. Modern bias: lean unchecked; checked exceptions don't compose with lambdas/streams. Full coverage in [04](./04_exceptions.md).

### Q20: What does `super` do in a constructor?
**A:** Calls a parent constructor. Must be the first statement in the child constructor. If you don't write it, the compiler inserts `super()` (no-arg). Used to thread arguments to the parent.

---

### Key Code Patterns

**Declare an interface with a default method**
```java
interface Repository<T> {
    Optional<T> findById(Long id);
    default boolean exists(Long id) { return findById(id).isPresent(); }   // shared default
}
```

**Record + compact constructor for validation**
```java
public record Email(String value) {
    public Email {
        if (value == null || !value.contains("@")) throw new IllegalArgumentException(value);
    }
}
```

**Sealed hierarchy + exhaustive switch**
```java
sealed interface Result<T> permits Success, Failure {}
record Success<T>(T value) implements Result<T> {}
record Failure<T>(String error) implements Result<T> {}
String describe(Result<?> r) { return switch (r) {
    case Success<?> s -> "ok: " + s.value();
    case Failure<?> f -> "err: " + f.error();
};}
```

**PECS in a generic API**
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) { /* … */ }
```

---

## 8. Self-Test

**Easy**
- [ ] What are the four pillars of OOP, in plain English?
- [ ] What's the difference between `==` and `.equals` for objects?
- [ ] What does `final` mean on a variable, a method, and a class?
- [ ] What does `static` mean?

**Medium**
- [ ] Walk through the Printer/ColorPrinter example: which method runs and why?
- [ ] When would you choose an interface over an abstract class? The reverse?
- [ ] What does PECS stand for and where does it apply?
- [ ] Why must you always override `hashCode` when you override `equals`?
- [ ] Build an immutable `Period(start, end, holidays)` value class.

**Hard**
- [ ] Explain type erasure and three runtime operations it makes impossible.
- [ ] Show how a sealed interface enables exhaustive switch — what does the compiler check?
- [ ] When does an inner class cause a memory leak, and how do you fix it?
- [ ] Explain how Java resolves the diamond problem with default methods.
- [ ] Why is `@Override` more than just documentation?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Class** | A blueprint for creating objects. |
| **Object** | A specific instance — real memory, real state. |
| **Interface** | A capability contract. No state, just method signatures (and since Java 8, default methods). |
| **Abstract class** | A partly-finished class — concrete bones plus blanks for subclasses. |
| **Encapsulation** | Hiding the wires: state private, behaviour exposed through controlled methods. |
| **Inheritance** | "is-a" relationship; subclass gets parent's stuff and may extend or replace it. |
| **Polymorphism** | One call site, many possible behaviours, picked by runtime type. |
| **Abstraction** | Specifying *what* without committing to *how*. |
| **Overloading** | Same name, different parameters; chosen at compile time. |
| **Overriding** | Subclass replaces parent method (same signature); chosen at runtime. |
| **Static dispatch** | The compiler picks the method (overloading). |
| **Dynamic / virtual dispatch** | The JVM picks the method via the vtable (overriding). |
| **Type erasure** | Generics are removed at runtime; `List<String>` is just `List`. |
| **PECS** | Producer Extends, Consumer Super — wildcard placement rule. |
| **Heap pollution** | A generic collection holding values whose runtime types violate the declared parameter. |
| **Liskov Substitution** | A subclass instance must work everywhere its parent works without breaking correctness. |
| **Composition** | Object holds another as a field; calls into it by reference (vs inheriting). |
| **Record** | Immutable data carrier with compiler-generated boilerplate (Java 16+). |
| **Sealed class** | Class/interface whose subclasses are restricted to a permitted list (Java 17+). |
| **Default method** | Concrete method body in an interface (Java 8+). |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/03_oop_language_doubts.md) · [← Prev: 02 Collections](./02_collections.md) · [Next: 04 Exceptions →](./04_exceptions.md)

[↑ Back to top](#03--oop--language-features)
