# 10 — Testing

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

---

## 🟢 Start Here — Testing in Plain English

### What is testing?

Testing is writing **code that checks your other code works correctly**. Instead of manually clicking through your app after every change, you write automated tests that do it for you in seconds.

```java
// Your code:
public int add(int a, int b) {
    return a + b;
}

// Your test:
@Test
void shouldAddTwoNumbers() {
    assertEquals(5, add(2, 3));    // expected: 5, actual: add(2,3)
    assertEquals(0, add(-1, 1));   // test edge case
}
// If add() is ever broken, this test catches it instantly.
```

### Why bother?

- **Confidence:** Change code without fear of breaking things — tests tell you immediately if something broke
- **Speed:** Run 500 tests in 2 seconds vs manually testing for 20 minutes
- **Documentation:** Tests show how your code is supposed to be used

### Three levels of testing

Think of testing a car:

| Level | What you test | Car analogy | Speed |
|-------|--------------|-------------|:-----:|
| **Unit test** | One class, in isolation | Test that one engine part works | ⚡ Fast |
| **Integration test** | Multiple parts working together | Engine + transmission + wheels together | 🐢 Medium |
| **E2E test** | The entire system | Take the car for a test drive | 🐌 Slow |

**The testing pyramid:** Write MANY unit tests, some integration tests, and few E2E tests.

### What's Mockito?

When testing a class, you don't want to use its real dependencies (real database, real email server). You use **mocks** — fake objects that pretend to be the real thing.

```java
// You want to test OrderService, but it uses a database.
// Don't test with a real database — too slow, too fragile.
// Instead, create a MOCK (fake) database:

OrderRepository mockRepo = mock(OrderRepository.class);         // fake database
when(mockRepo.findById("123")).thenReturn(new Order("123"));    // "when asked for 123, return this"

OrderService service = new OrderService(mockRepo);               // inject the fake
Order result = service.getOrder("123");                          // test your logic
assertEquals("123", result.getId());                             // works! no real DB needed.
```

> Key takeaway: Tests = code that checks your code. Unit tests = test one class (fast, many). Mocks = fake dependencies so you test in isolation.

---

## 📚 Study Material

### 1. Testing Pyramid

```
            ╱╲
           ╱  ╲          E2E / UI Tests (few)
          ╱    ╲         Slow, brittle, test full system
         ╱──────╲
        ╱        ╲       Integration Tests (some)
       ╱          ╲      Test component interactions (service + DB)
      ╱────────────╲
     ╱              ╲    Unit Tests (many)
    ╱                ╲   Fast, isolated, test one class
   ╱──────────────────╲
```

| Level | What | Speed | Example |
|-------|------|:-----:|---------|
| Unit | One class, mocked deps | ms | `OrderServiceTest` with mocked repo |
| Integration | Multiple components | sec | `@SpringBootTest` with embedded DB |
| E2E | Full system via HTTP/UI | sec-min | Selenium, RestAssured, Postman |

💡 **Goal:** Many fast unit tests, fewer integration tests, fewest E2E tests. If it fails, you immediately know what broke.

### 2. JUnit 5 — Lifecycle & Key Annotations

```java
class TradeServiceTest {
    
    @BeforeAll                                    // once before ALL tests (must be static)
    static void setupClass() { /* init shared resources */ }
    
    @BeforeEach                                   // before EACH test method
    void setUp() { service = new TradeService(mockRepo); }
    
    @Test
    @DisplayName("should reject negative quantity")
    void shouldRejectNegativeQuantity() {
        assertThrows(IllegalArgumentException.class,
            () -> service.createTrade("AAPL", -10));
    }
    
    @Test
    void shouldCalculateTotalCorrectly() {
        Trade trade = service.createTrade("AAPL", 100);
        
        assertAll("trade properties",                          // grouped assertions
            () -> assertEquals("AAPL", trade.getSymbol()),     // ALL run even if one fails
            () -> assertEquals(100, trade.getQuantity()),
            () -> assertNotNull(trade.getTimestamp())
        );
    }
    
    @ParameterizedTest                            // run same test with different inputs
    @CsvSource({"AAPL, 100", "GOOG, 50", "MSFT, 200"})
    void shouldCreateTradeForSymbol(String symbol, int qty) {
        Trade trade = service.createTrade(symbol, qty);
        assertEquals(symbol, trade.getSymbol());
    }
    
    @AfterEach                                    // after EACH test
    void tearDown() { /* cleanup */ }
    
    @AfterAll                                     // once after ALL tests (must be static)
    static void tearDownClass() { /* release shared resources */ }
}
```

**Key assertions:**
```java
assertEquals(expected, actual);                    // compare values
assertEquals(3.14, result, 0.001);                // floating point with delta
assertNotNull(obj);
assertTrue(condition);
assertFalse(condition);
assertThrows(Exception.class, () -> riskyCall()); // verify exception thrown
assertDoesNotThrow(() -> safeCall());
assertTimeout(Duration.ofSeconds(2), () -> slowCall());  // fails if too slow
assertAll(() -> ..., () -> ..., () -> ...);       // run all, report all failures
```

### 3. Mockito — Deep Dive

```java
@ExtendWith(MockitoExtension.class)               // JUnit 5 integration
class OrderServiceTest {
    
    @Mock                                          // creates mock (all methods return defaults)
    OrderRepository repo;
    
    @Mock
    NotificationService notifications;
    
    @InjectMocks                                   // creates OrderService, injects mocks
    OrderService service;
    
    @Test
    void shouldSaveOrder() {
        // ARRANGE: stub behaviour
        Order order = new Order("O001", "AAPL", 100);
        when(repo.save(any(Order.class)))           // when save is called with any Order
            .thenReturn(order);                      // return this order
        
        // ACT
        Order result = service.placeOrder(order);
        
        // ASSERT
        assertEquals("O001", result.getId());
        
        // VERIFY interactions
        verify(repo).save(order);                    // was save() called with this order?
        verify(repo, times(1)).save(any());          // called exactly once
        verify(notifications).orderPlaced(order);    // notification was sent
        verifyNoMoreInteractions(repo);              // no other calls to repo
    }
    
    @Test
    void shouldThrowWhenNotFound() {
        when(repo.findById("X"))
            .thenReturn(Optional.empty());
        
        assertThrows(ResourceNotFoundException.class,
            () -> service.getOrder("X"));
    }
    
    @Test
    void shouldCaptureArgument() {
        // ArgumentCaptor — inspect what was actually passed
        ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
        
        service.placeOrder(new Order("O001", "AAPL", 100));
        
        verify(repo).save(captor.capture());
        Order saved = captor.getValue();
        assertEquals("AAPL", saved.getSymbol());     // verify the exact object that was saved
    }
}
```

**Stubbing patterns:**
```java
when(mock.method()).thenReturn(value);               // return a value
when(mock.method()).thenThrow(new RuntimeException()); // throw
when(mock.method()).thenAnswer(invocation -> {        // custom logic
    String arg = invocation.getArgument(0);
    return arg.toUpperCase();
});
doNothing().when(mock).voidMethod();                 // stub void method
doThrow(new RuntimeException()).when(mock).voidMethod();
```

### 4. Mock vs Spy vs Fake

```java
// MOCK — fully fake, no real behaviour
OrderRepository mockRepo = mock(OrderRepository.class);
mockRepo.findById("X");  // returns null (default) unless stubbed

// SPY — wraps a real object, real methods run unless stubbed
OrderRepository realRepo = new InMemoryOrderRepository();
OrderRepository spyRepo = spy(realRepo);
spyRepo.findById("X");   // calls REAL findById
doReturn(mockOrder).when(spyRepo).findById("special");  // override specific call

// FAKE — real implementation, simplified (not a Mockito concept)
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new HashMap<>();
    public void save(Order o) { store.put(o.getId(), o); }
    public Optional<Order> findById(String id) { return Optional.ofNullable(store.get(id)); }
}
// Use fake when: you need realistic behaviour without a real database
```

### 5. Spring Boot Test Slices

```java
// Full context — loads everything (slow, use sparingly)
@SpringBootTest
class FullIntegrationTest {
    @Autowired OrderService service;
    @MockBean PaymentGateway gateway;   // replaces real bean with mock in context
}

// Web layer only — loads controllers, filters, advice (fast)
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService service;
    
    @Test
    void shouldReturn200() throws Exception {
        when(service.getOrder("O1")).thenReturn(new Order("O1"));
        
        mockMvc.perform(get("/api/orders/O1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value("O1"));
    }
}

// JPA layer only — loads repos + embedded DB (H2)
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repo;
    @Autowired TestEntityManager em;
    
    @Test
    void shouldFindBySymbol() {
        em.persist(new Order("AAPL", 100));
        List<Order> results = repo.findBySymbol("AAPL");
        assertEquals(1, results.size());
    }
}

// Use @ActiveProfiles("test") to load test-specific configuration
```

### 6. TDD — Red → Green → Refactor

```java
// 1. RED: Write a failing test FIRST
@Test
void shouldCalculateVWAP() {
    var calc = new VWAPCalculator();
    assertEquals(150.0, calc.calculate(trades), 0.01);  // fails — method doesn't exist yet
}

// 2. GREEN: Write MINIMAL code to pass
public class VWAPCalculator {
    public double calculate(List<Trade> trades) {
        double sumPriceQty = trades.stream().mapToDouble(t -> t.price() * t.qty()).sum();
        double sumQty = trades.stream().mapToDouble(Trade::qty).sum();
        return sumPriceQty / sumQty;
    }
}

// 3. REFACTOR: Clean up without changing behaviour (tests still pass)
// Extract, rename, simplify — tests are your safety net
```

### 7. Testing Best Practices

```java
// ✅ Test names describe behaviour, not method names
void shouldRejectOrderWhenInsufficientFunds()       // ✅ clear intent
void testProcessOrder()                             // ❌ vague

// ✅ Follow AAA: Arrange → Act → Assert
// ✅ One logical assertion per test (assertAll for related checks)
// ✅ Don't test private methods — test the public API that uses them
// ✅ Tests should be independent — no shared mutable state between tests
// ✅ Test edge cases: null, empty, boundary values, exceptions

// ❌ Don't test framework code (Spring, Hibernate) — test YOUR logic
// ❌ Don't mock types you don't own — use integration tests or fakes
// ❌ Don't verify internal implementation details — verify behaviour
```

---

## Rapid-Fire Q&A

### Q1: JUnit 5 — key annotations?
**A:** `@Test`, `@BeforeEach` / `@AfterEach` (per test), `@BeforeAll` / `@AfterAll` (per class, static), `@DisplayName`, `@Disabled`, `@ParameterizedTest` + `@ValueSource` / `@CsvSource`.

### Q2: Assertions — key ones?
**A:** `assertEquals(expected, actual)`, `assertNotNull(obj)`, `assertThrows(Exception.class, () -> ...)`, `assertTrue(condition)`, `assertAll(() -> ..., () -> ...)` for grouped assertions.

### Q3: What's Mockito and why use it?
**A:** Mocking framework. Create mock objects to isolate unit under test from dependencies. `@Mock` for mock, `@InjectMocks` for the class being tested. `when(mock.method()).thenReturn(value)` stubs behaviour. `verify(mock).method()` asserts interaction.

### Q4: `mock` vs `spy`?
**A:** `mock`: all methods return defaults (null, 0, false). Behaviour defined via `when`. `spy`: wraps a real object — real methods called unless explicitly stubbed. Use spy when you want mostly real behaviour with selective overrides.

### Q5: What are test doubles?
**A:** Dummy (passed, never used). Stub (returns canned answers). Mock (verifies interactions). Fake (working implementation, simplified — e.g., in-memory DB). Spy (real object with selective overrides).

### Q6: Unit vs Integration vs End-to-End testing?
**A:** Unit: test one class in isolation, mock dependencies. Fast. Integration: test multiple components together (e.g., service + DB). Spring `@SpringBootTest`. E2E: test the full system via HTTP/UI. Slow. Testing pyramid: many unit, fewer integration, fewest E2E.

### Q7: How do you test a private method?
**A:** You don't — test the public method that uses it. If the private method is complex enough to need direct testing, it's a code smell — extract it into its own class and test that.

### Q8: TDD — what is it?
**A:** Red → Green → Refactor. Write a failing test first, write minimal code to pass, refactor. Forces design-for-testability. Not always practical, but the discipline is valuable.

---

## Can you answer these cold?

- [ ] JUnit 5 lifecycle annotations — `@BeforeEach` vs `@BeforeAll`
- [ ] Mockito — `when/thenReturn`, `verify`, `@Mock` vs `@Spy`
- [ ] Testing pyramid — unit vs integration vs E2E
- [ ] How to test code that throws exceptions — `assertThrows`

[← Back to Index](./00_INDEX.md)
