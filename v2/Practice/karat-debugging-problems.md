# Karat Debugging & Dry-Run Problems (Java) — 20 Problems

Part 1 of the Carrot interview is "predict the output" / "find the bug" without code execution. You read code, dry-run it in your head, predict output, and explain the mechanism using exact technical vocabulary. If an exception is thrown, name the **exact** exception class.

**Rules:**
- No execution. Predict and explain.
- Aim for 25 seconds per problem.
- State *output* + *mechanism* + *exact exception class* (where applicable).
- Don't look at solutions until you've committed an answer.

## Index

| # | Title | Pattern Category |
|---|---|---|
| DBG-01 | Static List Accumulator | Shared mutable state |
| DBG-02 | Integer Cache Boundary | Autoboxing & Integer cache |
| DBG-03 | String Pool Reference | `==` vs `.equals` for Strings |
| DBG-04 | Map Lookup Auto-Unbox | NullPointerException on autoboxing |
| DBG-05 | String + Int Concatenation | Operator precedence in concat |
| DBG-06 | printf Format Mismatch | IllegalFormatConversionException |
| DBG-07 | Remove While Iterating | ConcurrentModificationException |
| DBG-08 | Lambda in Loop Capture | Effectively-final / closure |
| DBG-09 | Try-With-Resources Order | Reverse close order, suppressed exceptions |
| DBG-10 | Finally Overrides Return | Control flow in try/finally |
| DBG-11 | Switch Without Break | Fall-through |
| DBG-12 | Integer Division Truncation | Primitive arithmetic |
| DBG-13 | Floating-Point Equality | Float representation |
| DBG-14 | Shallow Copy of Nested List | Reference vs value copying |
| DBG-15 | Arrays.asList Add | UnsupportedOperationException |
| DBG-16 | Static Method "Override" | Method hiding vs overriding |
| DBG-17 | String split Trailing Empties | String.split(regex, limit) |
| DBG-18 | Stream Reduce From Loop | Stream API decoding |
| DBG-19 | final Keyword Three Ways | final on var/method/class |
| DBG-20 | Volatile Counter Race | Thread safety, atomicity vs visibility |

Bonus (worth knowing):

| # | Title | Pattern Category |
|---|---|---|
| DBG-21 | parseInt with Whitespace | NumberFormatException |
| DBG-22 | PECS Wildcard | Generics variance |
| DBG-23 | Mutable HashMap Key | hashCode contract / map key invariant |

---

## DBG-01 — Static List Accumulator

```java
public class Container {
    private static List<Integer> items = new ArrayList<>();
    
    public static List<Integer> add(Integer x) {
        items.add(x);
        return items;
    }
    
    public static void main(String[] args) {
        System.out.println(add(10));
        System.out.println(add(11));
        System.out.println(add(12));
    }
}
```

**Q:** What does this print? What's the underlying issue and how would you fix it?

**[Solution →](karat-solutions.md#dbg-01-solution)**

---

## DBG-02 — Integer Cache Boundary

```java
public class IntegerCache {
    public static void main(String[] args) {
        Integer a = 127, b = 127;
        Integer c = 128, d = 128;
        System.out.println(a == b);
        System.out.println(c == d);
        System.out.println(c.equals(d));
    }
}
```

**Q:** What does each line print? Why is the second different from the first?

**[Solution →](karat-solutions.md#dbg-02-solution)**

---

## DBG-03 — String Pool Reference

```java
public class StringPool {
    public static void main(String[] args) {
        String x = "hello";
        String y = "hello";
        String z = new String("hello");
        
        System.out.println(x == y);
        System.out.println(x == z);
        System.out.println(x.equals(z));
        System.out.println(x == z.intern());
    }
}
```

**Q:** Predict each line's output. Explain why `x == y` and `x == z` differ.

**[Solution →](karat-solutions.md#dbg-03-solution)**

---

## DBG-04 — Map Lookup Auto-Unbox

```java
public class MapUnbox {
    public static void main(String[] args) {
        Map<String, Integer> counts = new HashMap<>();
        counts.put("apple", 3);
        
        int appleCount = counts.get("apple");
        int bananaCount = counts.get("banana");
        
        System.out.println(appleCount + bananaCount);
    }
}
```

**Q:** What does this print? If it throws, name the exact exception class. How would you fix it idiomatically?

**[Solution →](karat-solutions.md#dbg-04-solution)**

---

## DBG-05 — String + Int Concatenation

```java
public class Concat {
    public static void main(String[] args) {
        System.out.println("Result: " + 5 + 3);
        System.out.println("Result: " + (5 + 3));
        System.out.println(5 + 3 + " items");
    }
}
```

**Q:** What does each line print? Explain the difference using the term *left-to-right associativity*.

**[Solution →](karat-solutions.md#dbg-05-solution)**

---

## DBG-06 — printf Format Mismatch

```java
public class Printf {
    public static void main(String[] args) {
        String name = "Alice";
        int age = 30;
        System.out.printf("Name: %s, Age: %d%n", name, age);
        System.out.printf("Name: %s, Age: %d%n", name, "30");
    }
}
```

**Q:** What does the first line print? What does the second do? Name the exact exception class.

**[Solution →](karat-solutions.md#dbg-06-solution)**

---

## DBG-07 — Remove While Iterating

```java
public class IteratingRemoval {
    public static void main(String[] args) {
        List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5));
        for (Integer n : nums) {
            if (n % 2 == 0) {
                nums.remove(n);
            }
        }
        System.out.println(nums);
    }
}
```

**Q:** What happens? Name the exact exception class. State two correct ways to fix it.

**[Solution →](karat-solutions.md#dbg-07-solution)**

---

## DBG-08 — Lambda in Loop Capture

```java
public class LambdaCapture {
    public static void main(String[] args) {
        List<Supplier<Integer>> funcs = new ArrayList<>();
        for (int i = 0; i < 3; i++) {
            int snapshot = i;
            funcs.add(() -> snapshot);
        }
        for (Supplier<Integer> f : funcs) {
            System.out.print(f.get() + " ");
        }
    }
}
```

**Q:** What does it print? Now: what would happen if you replaced `int snapshot = i` with using `i` directly inside the lambda? Why?

**[Solution →](karat-solutions.md#dbg-08-solution)**

---

## DBG-09 — Try-With-Resources Order

```java
public class Resources {
    static class Resource implements AutoCloseable {
        private final String name;
        Resource(String n) { this.name = n; System.out.println("Open " + name); }
        public void close() { System.out.println("Close " + name); }
    }
    
    public static void main(String[] args) {
        try (Resource a = new Resource("A");
             Resource b = new Resource("B")) {
            System.out.println("Use both");
        }
    }
}
```

**Q:** What is the output, in order? Why? What would happen if `b.close()` threw an exception while the try block was also throwing one?

**[Solution →](karat-solutions.md#dbg-09-solution)**

---

## DBG-10 — Finally Overrides Return

```java
public class FinallyReturn {
    public static int compute() {
        try {
            return 1;
        } finally {
            return 2;
        }
    }
    
    public static int compute2() {
        try {
            throw new RuntimeException("boom");
        } finally {
            return 99;
        }
    }
    
    public static void main(String[] args) {
        System.out.println(compute());
        System.out.println(compute2());
    }
}
```

**Q:** What does this print? Why is the `RuntimeException` from `compute2()` not propagated? Why is this an anti-pattern?

**[Solution →](karat-solutions.md#dbg-10-solution)**

---

## DBG-11 — Switch Without Break

```java
public class SwitchFallthrough {
    public static void main(String[] args) {
        int x = 2;
        switch (x) {
            case 1:
                System.out.println("one");
            case 2:
                System.out.println("two");
            case 3:
                System.out.println("three");
            default:
                System.out.println("other");
        }
    }
}
```

**Q:** What does it print? What's the term for this? How does Java 14+ switch expressions (`->` arrow) avoid it?

**[Solution →](karat-solutions.md#dbg-11-solution)**

---

## DBG-12 — Integer Division Truncation

```java
public class IntDiv {
    public static void main(String[] args) {
        int a = 5, b = 2;
        double r1 = a / b;
        double r2 = (double) a / b;
        double r3 = a / (double) b;
        System.out.println(r1 + " " + r2 + " " + r3);
    }
}
```

**Q:** What does this print? Why is `r1` not 2.5?

**[Solution →](karat-solutions.md#dbg-12-solution)**

---

## DBG-13 — Floating-Point Equality

```java
public class FloatEquality {
    public static void main(String[] args) {
        double a = 0.1 + 0.2;
        double b = 0.3;
        System.out.println(a);
        System.out.println(a == b);
        System.out.println(Math.abs(a - b) < 1e-9);
    }
}
```

**Q:** What does each line print? What's the correct way to compare doubles?

**[Solution →](karat-solutions.md#dbg-13-solution)**

---

## DBG-14 — Shallow Copy of Nested List

```java
public class ShallowDeep {
    public static void main(String[] args) {
        List<List<Integer>> original = new ArrayList<>();
        original.add(new ArrayList<>(List.of(1, 2)));
        original.add(new ArrayList<>(List.of(3, 4)));
        
        List<List<Integer>> shallow = new ArrayList<>(original);
        
        shallow.get(0).add(99);
        shallow.add(new ArrayList<>(List.of(5, 6)));
        
        System.out.println("original: " + original);
        System.out.println("shallow:  " + shallow);
    }
}
```

**Q:** Predict both lines of output. Explain the difference between *outer-list reference* and *inner-list reference*. How would you make a true deep copy?

**[Solution →](karat-solutions.md#dbg-14-solution)**

---

## DBG-15 — Arrays.asList Add

```java
public class FixedSizeList {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3);
        list.set(0, 99);
        System.out.println(list);
        list.add(4);
        System.out.println(list);
    }
}
```

**Q:** What is the output up to the first println? What happens at `list.add(4)`? Name the exact exception. Why does `set` work but `add` doesn't?

**[Solution →](karat-solutions.md#dbg-15-solution)**

---

## DBG-16 — Static Method "Override"

```java
public class StaticHiding {
    static class Animal {
        public static String sound() { return "generic"; }
    }
    
    static class Dog extends Animal {
        public static String sound() { return "bark"; }
    }
    
    public static void main(String[] args) {
        Animal a = new Dog();
        System.out.println(a.sound());
        
        Dog d = new Dog();
        System.out.println(d.sound());
    }
}
```

**Q:** What does this print? Explain why using the exact term *method hiding*. Why is `a.sound()` resolved differently than instance method calls?

**[Solution →](karat-solutions.md#dbg-16-solution)**

---

## DBG-17 — String split Trailing Empties

```java
public class SplitEdge {
    public static void main(String[] args) {
        System.out.println(Arrays.toString("a,b,c".split(",")));
        System.out.println(Arrays.toString("a,,b".split(",")));
        System.out.println(Arrays.toString("a,,".split(",")));
        System.out.println(Arrays.toString("a,,".split(",", -1)));
    }
}
```

**Q:** What does each line print? Why does line 3 differ from line 4?

**[Solution →](karat-solutions.md#dbg-17-solution)**

---

## DBG-18 — Stream Reduce From Loop

```java
public class StreamLoop {
    public static void main(String[] args) {
        List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);
        
        // Version A: stream
        int a = nums.stream()
            .filter(n -> n % 2 == 0)
            .mapToInt(n -> n * n)
            .sum();
        
        // Version B: equivalent loop
        int b = 0;
        for (int n : nums) {
            if (n % 2 == 0) {
                b += n * n;
            }
        }
        
        System.out.println(a + " " + b);
    }
}
```

**Q:** What does this print? Walk through the stream pipeline aloud, naming each operation by category (intermediate vs terminal).

**[Solution →](karat-solutions.md#dbg-18-solution)**

---

## DBG-19 — final Keyword Three Ways

```java
public class FinalThreeWays {
    final static int CONSTANT = 42;
    
    final static class Frozen {
        public String name = "fixed";
    }
    
    public final void doIt() {
        System.out.println("done");
    }
    
    public static void main(String[] args) {
        // Which of these compile? Which don't?
        // CONSTANT = 100;
        // class IceCold extends Frozen {}
        // (override doIt() in subclass)
        Frozen f = new Frozen();
        f.name = "changed";
        System.out.println(f.name);
    }
}
```

**Q:** Which three commented lines would fail to compile, and why each? Why does `f.name = "changed"` succeed even though `Frozen` is `final`?

**[Solution →](karat-solutions.md#dbg-19-solution)**

---

## DBG-20 — Volatile Counter Race

```java
public class VolatileCounter {
    private static volatile int count = 0;
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) count++;
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) count++;
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println(count);
    }
}
```

**Q:** Will this reliably print 20000? Why or why not? `volatile` guarantees what, and doesn't guarantee what? State two ways to fix it.

**[Solution →](karat-solutions.md#dbg-20-solution)**

---

## DBG-21 — parseInt with Whitespace

```java
public class ParseInt {
    public static void main(String[] args) {
        System.out.println(Integer.parseInt("42"));
        System.out.println(Integer.parseInt(" 42"));
        System.out.println(Integer.parseInt("42L"));
        System.out.println(Integer.parseInt(""));
    }
}
```

**Q:** Which lines succeed and which throw? Name the exact exception. What's the safer trim-then-parse pattern?

**[Solution →](karat-solutions.md#dbg-21-solution)**

---

## DBG-22 — PECS Wildcard

```java
public class Pecs {
    public static void main(String[] args) {
        List<Integer> ints = new ArrayList<>(List.of(1, 2, 3));
        List<? extends Number> nums = ints;
        
        // nums.add(4);              // Does this compile?
        // nums.add((Integer) 4);    // What about this?
        Number first = nums.get(0);  // Does this compile?
        
        List<? super Integer> sink = new ArrayList<Number>();
        sink.add(5);                 // Does this compile?
        // Integer x = sink.get(0);  // What about this?
        
        System.out.println(first);
    }
}
```

**Q:** For each commented line, state whether it compiles. Explain using the rule "Producer Extends, Consumer Super."

**[Solution →](karat-solutions.md#dbg-22-solution)**

---

## DBG-23 — Mutable HashMap Key

```java
public class MutableKey {
    public static void main(String[] args) {
        List<Integer> key = new ArrayList<>(List.of(1, 2, 3));
        Map<List<Integer>, String> map = new HashMap<>();
        map.put(key, "first");
        
        System.out.println(map.get(key));
        
        key.add(4);
        
        System.out.println(map.get(key));
        System.out.println(map.size());
    }
}
```

**Q:** Predict each println. Explain how the `hashCode`/`equals` contract gets violated. State the rule: "keys in a HashMap should be ____."

**[Solution →](karat-solutions.md#dbg-23-solution)**

---

## How to Use This Document

For each problem:

1. **Cover the question.** Read only the code.
2. **Predict the output aloud.** Don't write it down — say it.
3. **State the mechanism.** Use exact terminology — "Integer cache (-128 to 127)", "ConcurrentModificationException from `next()`", "method hiding rather than overriding", etc.
4. **Name the exception class** if one applies. The full class name, not "the iterator one."
5. **State the fix.** Even if not asked.

If any step takes more than 30 seconds, you don't have it solid. Re-cover, re-try after solutions review.

Solutions are in [`karat-solutions.md`](karat-solutions.md). Don't peek.

---

**← [Back to Master Index](karat-practice-INDEX.md)** | **Next: [Ladder Problems →](karat-ladder-problems.md)** | **[Solutions](karat-solutions.md)**
