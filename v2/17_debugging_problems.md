# Debugging Problems — Practice Set

[← Back to Index](./00_INDEX.md) | [Bug Taxonomy Reference](./16_debugging.md)

> **Instructions:** For each problem, follow the UMPIRE method:
> 1. **Read** the code — state what it's supposed to do
> 2. **Identify** the bug(s) — name the bug type from the taxonomy
> 3. **Fix** the code — write corrected version in the answer space
> 4. **Verify** — would your fix break any other behaviour?
>
> Leave your answers in the `Your Answer` sections. When ready, ask for evaluation.

---

## Problem 1 — The Vanishing Element

**Bug type hint:** Collections + equality

```java
public class UniqueNames {
    private final Set<String> names = new HashSet<>();

    public void addName(String name) {
        names.add(name);
    }

    public boolean hasName(String name) {
        return names.contains(name);
    }

    public static void main(String[] args) {
        UniqueNames un = new UniqueNames();
        un.addName(new String("Alice"));
        System.out.println(un.hasName("Alice"));           // What prints?
        System.out.println(un.hasName(new String("Alice"))); // What prints?
    }
}
```

**Question:** Does this code have a bug? What does it print and why?

### Your Answer
```
(write here)
```

---

## Problem 2 — The Broken Counter

**Bug type hint:** Concurrency

```java
public class RequestCounter {
    private volatile int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
// Used by 100 threads, each calling increment() 1000 times.
// Expected final count: 100,000
```

**Question:** Why is the final count less than 100,000? Name the bug type and fix it.

### Your Answer
```
(write here)
```

---

## Problem 3 — The Silent Failure

**Bug type hint:** Exception handling

```java
public class FileProcessor {
    public List<String> readLines(String path) {
        List<String> lines = new ArrayList<>();
        try {
            BufferedReader reader = new BufferedReader(new FileReader(path));
            String line;
            while ((line = reader.readLine()) != null) {
                lines.add(line);
            }
        } catch (IOException e) {
            // log and continue
        }
        return lines;
    }
}
```

**Question:** There are two bugs here. Find both, name the types, and fix.

### Your Answer
```
(write here)
```

---

## Problem 4 — The Infinite Loop

**Bug type hint:** Mutation during iteration

```java
public List<Integer> removeNegatives(List<Integer> numbers) {
    for (Integer num : numbers) {
        if (num < 0) {
            numbers.remove(num);
        }
    }
    return numbers;
}
```

**Question:** What happens when this runs with `[3, -1, 4, -5, 2]`? Fix it.

### Your Answer
```
(write here)
```

---

## Problem 5 — The Wrong Answer

**Bug type hint:** Off-by-one

```java
public static int binarySearch(int[] arr, int target) {
    int low = 0, high = arr.length;
    while (low < high) {
        int mid = (low + high) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) low = mid;
        else high = mid;
    }
    return -1;
}
```

**Question:** This binary search has two bugs. One can cause an infinite loop, the other can cause `ArrayIndexOutOfBoundsException`. Find and fix both.

### Your Answer
```
(write here)
```

---

## Problem 6 — The Race Condition

**Bug type hint:** Check-then-act

```java
public class UserCache {
    private final Map<String, User> cache = new HashMap<>();

    public User getUser(String id) {
        if (!cache.containsKey(id)) {
            User user = database.loadUser(id);
            cache.put(id, user);
        }
        return cache.get(id);
    }
}
// Called from multiple threads simultaneously
```

**Question:** There are two concurrency bugs. Name both and fix.

### Your Answer
```
(write here)
```

---

## Problem 7 — The Null Surprise

**Bug type hint:** Null handling + autoboxing

```java
public class ScoreCalculator {
    public int getHighestScore(Map<String, Integer> scores) {
        Integer highest = null;
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            if (highest == null || entry.getValue() > highest) {
                highest = entry.getValue();
            }
        }
        return highest;
    }
}
```

**Question:** What happens when `scores` is empty? What if a value in the map is `null`?

### Your Answer
```
(write here)
```

---

## Problem 8 — The Lost Update

**Bug type hint:** Concurrency — wrong synchronisation scope

```java
public class BankAccount {
    private int balance;

    public synchronized int getBalance() {
        return balance;
    }

    public synchronized void deposit(int amount) {
        balance += amount;
    }

    public void transfer(BankAccount to, int amount) {
        if (this.getBalance() >= amount) {
            this.deposit(-amount);
            to.deposit(amount);
        }
    }
}
```

**Question:** Even though `getBalance()` and `deposit()` are synchronized, `transfer()` has a race condition. Explain why and fix it.

### Your Answer
```
(write here)
```

---

## Problem 9 — The Wrong Comparison

**Bug type hint:** Wrong operator

```java
public class StatusChecker {
    public boolean isActive(String status) {
        return status == "ACTIVE";
    }
}
// Called with: isActive(user.getStatus())  where getStatus() returns "ACTIVE"
```

**Question:** Why does this sometimes return `false` even when the status is "ACTIVE"? Fix it.

### Your Answer
```
(write here)
```

---

## Problem 10 — The Deadlock

**Bug type hint:** Concurrency — lock ordering

```java
public class TransferService {
    public void transfer(Account from, Account to, BigDecimal amount) {
        synchronized (from) {
            synchronized (to) {
                from.debit(amount);
                to.credit(amount);
            }
        }
    }
}
// Thread 1: transfer(accountA, accountB, 100)
// Thread 2: transfer(accountB, accountA, 50)
```

**Question:** Why does this deadlock? Fix it.

### Your Answer
```
(write here)
```

---

## Problem 11 — The Stale Cache

**Bug type hint:** Visibility

```java
public class FeatureFlag {
    private boolean enabled = false;

    public void enable()  { enabled = true;  }
    public void disable() { enabled = false; }

    public void processIfEnabled(Request request) {
        if (enabled) {
            process(request);
        }
    }
}
// enable() called from admin thread
// processIfEnabled() called from 50 request-handler threads
```

**Question:** Why might request threads never see `enabled = true`? Fix it.

### Your Answer
```
(write here)
```

---

## Problem 12 — The Overflow

**Bug type hint:** Integer overflow

```java
public class ArrayMerger {
    public static int[] merge(int[] a, int[] b) {
        int[] result = new int[a.length + b.length];
        // merge logic...
        return result;
    }

    public static long sumAll(int[] arr) {
        int sum = 0;
        for (int val : arr) {
            sum += val;
        }
        return sum;
    }
}
// Called with arrays containing values up to Integer.MAX_VALUE
```

**Question:** `sumAll` returns a `long` but the sum can still overflow. Why? Fix it.

### Your Answer
```
(write here)
```

---

## Problem 13 — The Spurious Wakeup

**Bug type hint:** Concurrency — wait/notify

```java
public class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedBuffer(int capacity) { this.capacity = capacity; }

    public synchronized void put(T item) throws InterruptedException {
        if (queue.size() == capacity) {
            wait();
        }
        queue.add(item);
        notifyAll();
    }

    public synchronized T take() throws InterruptedException {
        if (queue.isEmpty()) {
            wait();
        }
        T item = queue.poll();
        notifyAll();
        return item;
    }
}
```

**Question:** There's a classic bug in both `put` and `take`. What is it and why does it matter?

### Your Answer
```
(write here)
```

---

## Problem 14 — The Broken Equals

**Bug type hint:** equals/hashCode contract

```java
public class Employee {
    private String name;
    private int id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee e = (Employee) o;
        return id == e.id && Objects.equals(name, e.name);
    }
    // No hashCode override
}
// Used as key in HashMap
```

**Question:** What happens when you `put` an Employee into a HashMap and then try to `get` it with an equal Employee? Why?

### Your Answer
```
(write here)
```

---

## Problem 15 — The Floating Point Trap

**Bug type hint:** Floating point comparison

```java
public class PriceCalculator {
    public boolean isPriceMatch(double price1, double price2) {
        return price1 == price2;
    }

    public double calculateTotal(int quantity, double unitPrice) {
        double total = 0;
        for (int i = 0; i < quantity; i++) {
            total += unitPrice;
        }
        return total;
    }
}
// calculateTotal(10, 0.1) should return 1.0
// isPriceMatch(calculateTotal(10, 0.1), 1.0) returns false
```

**Question:** Why does the price match fail? What are two different fixes?

### Your Answer
```
(write here)
```

---

## Problem 16 — The ThreadLocal Leak

**Bug type hint:** Resource leak in thread pool

```java
public class RequestContext {
    private static final ThreadLocal<String> userId = new ThreadLocal<>();

    public static void set(String id) { userId.set(id); }
    public static String get()        { return userId.get(); }
}

// In a servlet filter:
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) 
        throws IOException, ServletException {
    RequestContext.set(extractUserId(req));
    chain.doFilter(req, res);
}
```

**Question:** In a thread pool (like Tomcat's), this leaks. Explain why and fix it.

### Your Answer
```
(write here)
```

---

## Problem 17 — The Wrong Stream Result

**Bug type hint:** Stream API misuse

```java
public class OrderService {
    public Optional<Order> findMostExpensive(List<Order> orders) {
        return orders.stream()
            .sorted(Comparator.comparing(Order::getTotal).reversed())
            .findFirst();
    }

    public Map<String, List<Order>> groupByStatus(List<Order> orders) {
        Map<String, List<Order>> result = orders.stream()
            .collect(Collectors.groupingBy(Order::getStatus));

        // "PENDING" should always be in the map, even if empty
        return result;
    }
}
```

**Question:** `findMostExpensive` works but is inefficient. `groupByStatus` has a missing-key bug. Fix both.

### Your Answer
```
(write here)
```

---

## Problem 18 — The Singleton That Isn't

**Bug type hint:** Concurrency — DCL

```java
public class AppConfig {
    private static AppConfig instance;

    private AppConfig() { /* expensive init */ }

    public static AppConfig getInstance() {
        if (instance == null) {
            synchronized (AppConfig.class) {
                if (instance == null) {
                    instance = new AppConfig();
                }
            }
        }
        return instance;
    }
}
```

**Question:** This double-checked locking has a subtle bug. What is it? (Hint: it involves instruction reordering.)

### Your Answer
```
(write here)
```

---

## Problem 19 — The Broken Pagination

**Bug type hint:** Off-by-one + edge case

```java
public class Paginator<T> {
    public List<T> getPage(List<T> items, int pageNumber, int pageSize) {
        int start = pageNumber * pageSize;
        int end = start + pageSize;
        return items.subList(start, end);
    }
}
// getPage(items, 0, 10) — first page
// getPage(items, 5, 10) — sixth page
// items has 53 elements, pageSize = 10
// getPage(items, 5, 10) should return 3 elements (indices 50-52)
```

**Question:** What happens on the last page? What happens if `pageNumber` is too large? Fix both issues.

### Your Answer
```
(write here)
```

---

## Problem 20 — The Transaction Trap

**Bug type hint:** Spring proxy + self-invocation

```java
@Service
public class PaymentService {

    @Transactional
    public void processPayment(Payment payment) {
        validatePayment(payment);
        debitAccount(payment);
        creditAccount(payment);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditPayment(Payment payment) {
        auditRepo.save(new AuditEntry(payment));
    }

    public void processAndAudit(Payment payment) {
        processPayment(payment);
        auditPayment(payment);
    }
}
// Called from controller: paymentService.processAndAudit(payment);
```

**Question:** `processAndAudit` calls two `@Transactional` methods. Do the transactional annotations work? Why or why not? What happens if `creditAccount` throws an exception — is the audit saved?

### Your Answer
```
(write here)
```

---

## Scoring Guide (for self-evaluation)

After attempting all 20, evaluate yourself:

| Score | Meaning |
|:-----:|---------|
| **3** | Named the bug type correctly + wrote correct fix + mentioned edge cases |
| **2** | Found the bug + wrote a fix, but missed edge cases or used wrong terminology |
| **1** | Found the bug but fix was incomplete or wrong |
| **0** | Didn't find the bug or misidentified it |

| Total | Assessment |
|:-----:|-----------|
| 50-60 | Ready for Karat |
| 40-49 | Review weak areas, retry misses |
| 30-39 | Revisit v2 study material, especially concurrency + collections |
| <30 | More study time needed — focus on bug taxonomy in `16_debugging.md` |

---

## Bug Type Coverage

| # | Bug Type | Problems |
|:-:|----------|:--------:|
| 1 | Off-by-one | 5, 19 |
| 2 | Null handling | 7 |
| 3 | Wrong operator | 9, 15 |
| 4 | Mutation during iteration | 4 |
| 5 | Integer overflow | 12 |
| 6 | Resource leak | 3, 16 |
| 7 | Check-then-act race | 6, 8 |
| 8 | Visibility | 11 |
| 9 | Deadlock | 10 |
| 10 | Spurious wakeup | 13 |
| 11 | equals/hashCode | 1, 14 |
| 12 | Floating point | 15 |
| 13 | Spring proxy | 20 |
| 14 | Stream misuse | 17 |
| 15 | DCL / volatile | 18 |
| 16 | Volatile + atomicity | 2 |

[← Back to Index](./00_INDEX.md)
