# 10 — Testing

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/10_testing_doubts.md) · [← Prev: 09 SOLID](./09_solid.md) · [Next: 11 Spring Core →](./11_spring_core.md)
>
> **Priority:** 🟢 Medium · **Related topics:** [04 Exceptions](./04_exceptions.md) · [11 Spring Core](./11_spring_core.md) · [12 Spring Boot](./12_spring_boot.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — why test
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — anatomy of a unit test
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — JUnit + Mockito in one screen
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 The testing pyramid](#41-the-testing-pyramid)
   - [4.2 JUnit 5 lifecycle](#42-junit-5-lifecycle--key-annotations)
   - [4.3 Assertions](#43-assertions--what-to-reach-for)
   - [4.4 Mockito basics](#44-mockito-basics)
   - [4.5 Mock vs Spy vs Fake](#45-mock-vs-spy-vs-fake)
   - [4.6 Spring Boot test slices](#46-spring-boot-test-slices)
   - [4.7 TDD red-green-refactor](#47-tdd--red--green--refactor)
   - [4.8 Best-practice rules](#48-best-practice-rules)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 Argument captors and verification](#51-argument-captors-and-verification)
   - [5.2 Stubbing strategies](#52-stubbing-strategies-thenreturn-thenanswer-thenthrow)
   - [5.3 Static / final / private mocking](#53-static--final--private-method-mocking)
   - [5.4 Test containers vs embedded DB](#54-test-containers-vs-embedded-db)
   - [5.5 Flaky tests](#55-flaky-tests-and-how-to-fix-them)
   - [5.6 Coverage vs confidence](#56-coverage-vs-confidence)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

You're building a car. You can:

- **Manually drive it after every change** to confirm it still works. Slow, error-prone, doesn't scale. Eventually you stop checking everything; eventually it breaks in production.
- **Build a test rig** that runs every check automatically — start the engine, check the brakes, verify the doors close. Run it in two seconds before every release.

Tests are the rig. They protect you from the "I changed something small and didn't realise it broke something else" problem — the single biggest source of bugs in long-lived codebases.

Tests give you three things:

1. **Confidence.** Refactor freely; if you broke something, the test rig tells you in seconds.
2. **Documentation.** A test named `shouldRollBackOnInsufficientFunds` is clearer than a paragraph of comments.
3. **Design pressure.** Code that's hard to test is usually badly designed (too many dependencies, hidden globals). Tests force you toward better structure.

Three levels matter:

- **Unit tests** — one class, dependencies replaced with **mocks** (fakes). Milliseconds. The bulk of your test suite.
- **Integration tests** — multiple components together (service + DB, controller + service). Seconds. Catches wiring problems.
- **End-to-end tests** — entire app via HTTP/UI. Slow, brittle, you keep them few. Catches user-facing regressions.

> The shape that works is the **testing pyramid**: many fast unit tests at the base, fewer integration tests in the middle, a thin layer of E2E at the top. Inverted pyramids (lots of E2E, few unit) are slow, flaky, and hard to debug — the test fails but you don't know which class broke.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

A unit test always follows three beats: **Arrange → Act → Assert**.

You're testing `OrderService.placeOrder(order)`. It calls `repo.save(order)` and `notifications.orderCreated(order)`. You don't want a real database or a real email server — just check that `placeOrder` does the right things in the right order.

```
ARRANGE
    repo = mock OrderRepository       (a fake)
    notifications = mock NotificationService
    when repo.save(any) → return the order back
    service = new OrderService(repo, notifications)

ACT
    result = service.placeOrder(order)

ASSERT
    result.id == order.id              (return value correct)
    verify repo.save called once with the order
    verify notifications.orderCreated called once with the order
    verify nothing else happened on either mock
```

What this walkthrough makes obvious:

1. The **test's job is to isolate the unit**. Real collaborators are replaced by mocks; only `OrderService`'s logic is exercised.
2. Two kinds of assertion exist: **state assertions** (return value, fields) and **interaction assertions** (`verify(...)` — was the right method called with the right arguments?). Both are legitimate. Use state when you can; reach for interactions when the unit's job is to *coordinate* its collaborators.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.*;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)              // JUnit 5 + Mockito integration
class OrderServiceTest {

    @Mock OrderRepository repo;                   // auto-created mock
    @Mock NotificationService notifications;      // auto-created mock
    @InjectMocks OrderService service;            // creates OrderService, injects @Mocks via ctor

    @Test
    @DisplayName("placeOrder saves and notifies — happy path")
    void placeOrder_happyPath() {
        // ARRANGE
        var order = new Order("O001", "AAPL", 100);
        when(repo.save(any(Order.class))).thenReturn(order);   // stub repo.save

        // ACT
        var result = service.placeOrder(order);

        // ASSERT — state
        assertEquals("O001", result.id());

        // ASSERT — interactions
        verify(repo).save(order);                              // saved exactly this order
        verify(notifications).orderCreated(order);             // notification fired
        verifyNoMoreInteractions(repo, notifications);         // nothing else
    }

    @Test
    void placeOrder_throwsWhenSaveFails() {
        var order = new Order("O001", "AAPL", 100);
        when(repo.save(any())).thenThrow(new DataAccessException("DB down"));

        assertThrows(DataAccessException.class, () -> service.placeOrder(order));
        verify(notifications, never()).orderCreated(any());    // no notification on failure
    }
}
```

What just happened: `@Mock` and `@InjectMocks` build the unit and its fake collaborators automatically. `when(...).thenReturn(...)` programs the mock. The actual call to `service.placeOrder` runs the real `OrderService` logic. We assert the return value (state) and the interactions with the mocks. Two tests cover the happy path and one failure path — the bare minimum for a method like this.

---

## 4. Build Up — Practical Patterns

### 4.1 The testing pyramid

```
            ╱╲
           ╱  ╲          E2E (few)        — full app via HTTP/UI; slow, brittle
          ╱    ╲         seconds-minutes
         ╱──────╲
        ╱        ╲       Integration       — multiple components; seconds
       ╱          ╲      service + DB, controller + service
      ╱────────────╲
     ╱              ╲    Unit (many)       — one class isolated; milliseconds
    ╱                ╲   bulk of the suite
   ╱──────────────────╲
```

| Level | What | Speed | Tool examples |
|---|---|---|---|
| Unit | One class, dependencies mocked | ms | JUnit + Mockito |
| Integration | Multiple beans wired together | sec | `@SpringBootTest`, `@DataJpaTest`, Testcontainers |
| E2E | Full HTTP/UI workflow | sec–min | RestAssured, Postman, Selenium, Playwright |

Goal: **many unit tests** at the base. When something breaks, the failed test name tells you the responsible class.

### 4.2 JUnit 5 lifecycle + key annotations

```java
class TradeServiceTest {

    @BeforeAll  static void initShared()     { /* once before ALL tests, must be static */ }
    @BeforeEach void  setUp()                { /* before EACH test */ }
    @AfterEach  void  tearDown()             { /* after EACH test */ }
    @AfterAll   static void teardownShared() { /* once after ALL tests, must be static */ }

    @Test
    @DisplayName("rejects negative quantity")
    void rejectsNegative() {
        assertThrows(IllegalArgumentException.class,
            () -> service.createTrade("AAPL", -10));
    }

    @ParameterizedTest                      // run the same test with different inputs
    @CsvSource({"AAPL, 100", "GOOG, 50", "MSFT, 200"})
    void createsTrade(String symbol, int qty) {
        var t = service.createTrade(symbol, qty);
        assertEquals(symbol, t.symbol());
    }

    @Test @Disabled("flaky — see TICKET-123")
    void temporaryDisable() { /* skipped */ }
}
```

Key annotations: `@Test`, `@BeforeEach`/`@AfterEach`, `@BeforeAll`/`@AfterAll` (must be `static`), `@DisplayName`, `@Disabled`, `@ParameterizedTest` + `@ValueSource` / `@CsvSource` / `@EnumSource`.

### 4.3 Assertions — what to reach for

```java
// State / value
assertEquals(expected, actual);
assertEquals(3.14, x, 0.001);                                  // floating point with delta
assertNotNull(obj);
assertTrue(condition); assertFalse(condition);

// Exceptions
var ex = assertThrows(IllegalArgumentException.class, () -> service.boom());
assertEquals("bad", ex.getMessage());                          // assert on the exception

// Doesn't throw / completes within timeout
assertDoesNotThrow(() -> service.fine());
assertTimeout(Duration.ofSeconds(2), () -> service.slow());    // fails if too slow

// Group related assertions — ALL run, ALL failures reported
assertAll("trade fields",
    () -> assertEquals("AAPL", t.symbol()),
    () -> assertEquals(100,    t.qty()),
    () -> assertNotNull(t.timestamp())
);

// AssertJ (richer alternative — most teams use it)
import static org.assertj.core.api.Assertions.assertThat;
assertThat(orders).hasSize(3).extracting(Order::id).containsExactly("O1", "O2", "O3");
```

> 💡 AssertJ's fluent API tends to read better than JUnit's `assertEquals`. Most modern Java teams use it.

### 4.4 Mockito basics

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;          // creates a mock
    @Mock NotificationService notes;
    @InjectMocks OrderService service;   // creates OrderService, injects mocks via ctor

    @Test
    void shouldSave() {
        // ARRANGE — stub
        var order = new Order("O001");
        when(repo.save(any(Order.class))).thenReturn(order);

        // ACT
        var result = service.placeOrder(order);

        // ASSERT
        assertEquals("O001", result.id());

        // VERIFY interactions
        verify(repo).save(order);                   // called with this argument
        verify(repo, times(1)).save(any());         // called exactly once
        verify(notes).orderCreated(order);
        verifyNoMoreInteractions(repo, notes);      // nothing else
    }

    @Test
    void shouldThrowWhenNotFound() {
        when(repo.findById("X")).thenReturn(Optional.empty());
        assertThrows(NotFoundException.class, () -> service.getOrder("X"));
    }
}
```

**Argument matchers** — `any()`, `eq(value)`, `argThat(predicate)`. If you use a matcher in any position of a stubbing call, you must use matchers in *every* position.

### 4.5 Mock vs Spy vs Fake

| | What it is | Behaviour |
|---|---|---|
| **Mock** | Fully fake — Mockito-generated | All methods return defaults (null/0/false) until stubbed |
| **Spy** | Wraps a real instance | Real methods run unless stubbed; `doReturn()`/`doThrow()` to override |
| **Fake** | Real, simplified implementation (e.g. in-memory map repo) | You write it once; behaves like the real thing for test purposes |
| **Stub** | An object returning canned answers | Same as a mock with stubbings; no interaction verification |
| **Dummy** | Object passed but never used | Just to fill a parameter slot |

```java
// SPY — wrap a real object, override one specific call
OrderRepository real = new InMemoryOrderRepository();
OrderRepository spy = spy(real);
doReturn(specialOrder).when(spy).findById("special");  // override only this call
spy.findById("normal");                                 // calls real impl

// FAKE — your own minimal implementation, useful when:
//   - the real impl is too slow (real DB)
//   - you need realistic behaviour (pagination, transactions) that mocks can't replicate
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new HashMap<>();
    public void save(Order o) { store.put(o.id(), o); }
    public Optional<Order> findById(String id) { return Optional.ofNullable(store.get(id)); }
}
```

### 4.6 Spring Boot test slices

```java
// FULL CONTEXT — slow, integration tests
@SpringBootTest
class OrderIntegrationTest {
    @Autowired OrderService service;
    @MockBean PaymentGateway gateway;       // replaces real bean with a mock in the context
}

// WEB LAYER ONLY — fast, controller tests
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService service;

    @Test void getsOrder() throws Exception {
        when(service.getOrder("O1")).thenReturn(new Order("O1"));
        mockMvc.perform(get("/api/orders/O1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value("O1"));
    }
}

// JPA LAYER ONLY — embedded DB, auto-rollback
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repo;
    @Autowired TestEntityManager em;
}

// Other slices: @JsonTest (Jackson), @WebFluxTest (reactive), @JdbcTest (JDBC), @DataMongoTest, @RestClientTest
```

### 4.7 TDD — red → green → refactor

A discipline, not a religion:

1. **RED** — write a failing test. The test names what you want and verifies it doesn't yet work.
2. **GREEN** — write the minimum code to make the test pass. Don't perfect it.
3. **REFACTOR** — clean up under the safety net of the green test. Rename, extract, simplify. Tests stay green.

When TDD shines: well-defined units (parsers, calculators, validators). When it doesn't: exploratory work, GUI fiddling, integration with unfamiliar APIs.

### 4.8 Best-practice rules

```java
// ✅ DO
void shouldRollBackOnInsufficientFunds()    // describe behaviour, not method name
// AAA structure: Arrange → Act → Assert (or Given/When/Then)
// One logical concept per test (use assertAll for related fields)
// Test edge cases: null, empty, boundary, concurrent
// Independent tests — no shared mutable state between them

// ❌ DON'T
void testProcessOrder()                      // vague — what does success look like?
catch (Exception e) {}                       // swallowing in tests hides real failures
when(myFinalClass.method())                  // mocking final classes (without inline mock-maker)
verify(repo, times(1)).save(any())           // over-verifying makes refactor painful
@Test private void noisyTest()              // private tests aren't run; never use `private`
```

---

## 5. Going Deep — Interview-Level Material

### 5.1 Argument captors and verification

When the unit *transforms* the input before passing it on, plain `verify(...)` isn't enough. Use an `ArgumentCaptor` to inspect what actually got passed:

```java
@Test
void shouldEnrichOrderBeforeSaving() {
    var input = new Order("O1", "AAPL", 100);
    var captor = ArgumentCaptor.forClass(Order.class);

    service.placeOrder(input);

    verify(repo).save(captor.capture());
    var saved = captor.getValue();
    assertEquals("AAPL", saved.symbol());
    assertNotNull(saved.timestamp());                // service should have set this
}
```

Captors also let you assert on *all* invocations: `captor.getAllValues()` returns the list.

### 5.2 Stubbing strategies: `thenReturn`, `thenAnswer`, `thenThrow`

```java
when(mock.method()).thenReturn(value);                          // return a value
when(mock.method()).thenReturn(v1, v2, v3);                     // first call → v1, second → v2, third onward → v3
when(mock.method()).thenThrow(new RuntimeException());          // throw

// thenAnswer — compute the return based on the actual arguments
when(repo.findById(anyString())).thenAnswer(invocation -> {
    String id = invocation.getArgument(0);
    return Optional.of(new Order(id, "AAPL", 100));
});

// Stubbing void methods — different syntax
doNothing().when(mock).voidMethod();
doThrow(new RuntimeException()).when(mock).voidMethod();
doAnswer(...).when(mock).voidMethod();

// doReturn — for spies (when() runs the real method, doReturn() doesn't)
OrderRepository spy = spy(realRepo);
doReturn(special).when(spy).findById("X");           // safe even if real findById throws
```

### 5.3 Static / final / private method mocking

Mockito (≥ 3.4) supports static-method mocking via `mockStatic`, but it's a sign of bad design — a class that calls `Foo.staticMethod()` is hard to test because the dependency is implicit. **Prefer to refactor**: inject a dependency, wrap the static call.

```java
try (MockedStatic<Files> mocked = mockStatic(Files.class)) {
    mocked.when(() -> Files.readAllBytes(any())).thenReturn(new byte[]{1, 2});
    // … test runs here …
}
// outside the try: original Files.* behaviour restored
```

`final` classes are mockable since Mockito 2 with the inline mock-maker. `private` methods are mockable in theory but **never test private methods directly** — test the public method that uses them.

### 5.4 Test containers vs embedded DB

| | What | When |
|---|---|---|
| **Embedded DB** (H2) | An in-process DB pretending to be Postgres/MySQL | Fast unit/integration with simple SQL |
| **Testcontainers** | Real Postgres/MySQL/Kafka/etc. running in Docker, started for the test | When SQL dialect, extensions, or behaviour matters |

Embedded DBs are faster to start but lie about behaviour — H2 in PostgreSQL mode silently differs from real Postgres. **Testcontainers** gives you the real database, real version, real behaviour. The standard for serious integration tests.

```java
@Testcontainers
class IntegrationTest {
    @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
}
```

### 5.5 Flaky tests and how to fix them

A test that sometimes passes and sometimes fails. Flaky tests destroy trust in the suite — people stop reading failures, then real bugs ship. Common causes:

| Cause | Fix |
|---|---|
| Timing / sleeps | Replace `Thread.sleep` with **Awaitility** (poll-and-timeout) or proper synchronisation |
| Test order dependence | Tests share mutable state; isolate or reset between tests |
| Real network/clock | Inject `Clock`; use `WireMock` or stubs for HTTP |
| Concurrency | Avoid testing the *thread scheduler*; test the protocol with a deterministic harness |
| Random data | Seed all randomness; or generate but log the seed for reproduction |
| Resource cleanup | Tests leak files/connections; use try-with-resources, ensure tearDown is correct |

**Rule:** a flaky test is worse than no test. Either fix it within a sprint or delete it. Don't `@Disabled` indefinitely.

### 5.6 Coverage vs confidence

Line / branch coverage is a **floor**, not a ceiling. 100% coverage means "every line ran"; it doesn't mean "every behaviour is tested". You can write tests that hit every line and verify nothing.

What matters: **scenarios you'd be embarrassed to have ship broken.** Focus on:

- The happy path.
- The two or three most common error paths (insufficient funds, invalid input, downstream timeout).
- Boundary conditions (empty list, single item, max size).
- The hard-won bug fixes (one regression test per bug closed).

Coverage tools (JaCoCo, IntelliJ) help you find untested code, but the metric is a hint, not a goal.

---

## 6. Memory Aids

### Decision tree: "what kind of test do I need?"

```
Is this one method's logic, with collaborators that are easy to mock?
└── UNIT TEST — JUnit + Mockito.

Does it cross a Spring boundary (controller → service → repo) or hit JSON / SQL?
└── INTEGRATION TEST.
    ├── Just the controller layer? → @WebMvcTest with @MockBean services.
    ├── Just the JPA layer? → @DataJpaTest with embedded H2 or Testcontainers.
    └── Multiple layers wired together? → @SpringBootTest.

Does it test full user-visible behaviour through HTTP/UI?
└── E2E — RestAssured / Selenium. Few of these.
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "What is a unit test?" | Isolated single class | "JUnit + Mockito. Replace dependencies with mocks. Milliseconds." |
| "Mock vs Spy?" | All-fake vs wrap-real | "Mock returns defaults until stubbed. Spy calls the real method unless you override." |
| "When over-mock?" | Mocking what you don't own / value types | "Mock interfaces of YOUR collaborators. Don't mock value types or third-party classes." |
| "Why constructor injection helps testing?" | Pass mocks via constructor | "No reflection, no Spring needed for the test." |
| "How to test an exception path?" | `assertThrows` | "Returns the thrown exception so you can assert on its message." |
| "What's a flaky test?" | Inconsistent result | "Worst kind of debt. Fix the root cause (timing, order, concurrency) or delete it." |
| "100% coverage = good tests?" | Coverage is a floor | "Lines covered ≠ behaviours tested. Coverage is a hint, not a goal." |
| "When use Testcontainers?" | Real DB behaviour | "When SQL dialect or DB-specific behaviour matters. H2 lies." |

### Three anchor pictures

1. **Pyramid: many unit, few E2E.** Inverted = slow + flaky.
2. **Arrange / Act / Assert.** Every test has these three beats.
3. **Mock interfaces of your collaborators.** Don't mock value types, framework internals, or third-party APIs.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: JUnit 5 — key annotations?
**A:** `@Test`, `@BeforeEach` / `@AfterEach`, `@BeforeAll` / `@AfterAll` (must be static), `@DisplayName`, `@Disabled`, `@ParameterizedTest` + `@ValueSource` / `@CsvSource` / `@MethodSource`.

### Q2: Key assertions?
**A:** `assertEquals`, `assertNotNull`, `assertTrue`, `assertThrows(Exception.class, () -> ...)`, `assertAll(...)` for grouped assertions, `assertTimeout(Duration, () -> ...)` for time bounds. AssertJ's `assertThat(...).contains/hasSize/extracting(...)` is richer.

### Q3: What's Mockito and why use it?
**A:** Mocking framework. Replaces real dependencies with controllable fakes so you can test a unit in isolation. `@Mock` creates a mock, `@InjectMocks` creates the unit and injects mocks via constructor. `when(...).thenReturn(...)` stubs; `verify(...)` asserts interactions.

### Q4: `mock` vs `spy`?
**A:** Mock — fully fake; methods return defaults until stubbed. Spy — wraps a real instance; real methods run unless you override with `doReturn`/`doThrow`. Spies are useful for legacy code where you want mostly real behaviour with selective overrides.

### Q5: What are test doubles?
**A:** Dummy (passed but unused), Stub (canned answers), Mock (verifies interactions), Fake (real but simplified — e.g. in-memory repo), Spy (real with selective overrides). Mockito covers all five depending on how you use it.

### Q6: Unit vs Integration vs E2E?
**A:** Unit — one class isolated, mocks for collaborators (ms). Integration — multiple components, real wiring, sometimes embedded DB (sec). E2E — full app via HTTP/UI (sec–min). Pyramid: many unit, fewer integration, fewest E2E.

### Q7: How do you test a private method?
**A:** You don't. Test the public method that uses it. If a private method is complex enough to need direct testing, extract it into its own class with a public API. Reaching for reflection to test privates is a code smell.

### Q8: What's TDD?
**A:** Red → Green → Refactor. Write a failing test first, write minimal code to pass, refactor under the safety net. Forces design-for-testability. Strong discipline for well-defined units; awkward for exploratory work.

### Q9: `@MockBean` vs `@Mock`?
**A:** `@Mock` (Mockito) — pure unit test, no Spring context. `@MockBean` (Spring Boot) — replaces a real bean with a Mockito mock *inside the loaded Spring context*. Used in `@SpringBootTest`, `@WebMvcTest`, etc.

### Q10: Spring Boot test slices?
**A:** `@SpringBootTest` (full context, slow), `@WebMvcTest(Controller.class)` (web layer only, fast), `@DataJpaTest` (JPA layer + embedded DB, auto-rollback), `@JsonTest` (Jackson), `@WebFluxTest` (reactive), `@JdbcTest` (JDBC).

### Q11: How do you handle a test that's flaky?
**A:** Find the root cause: timing (replace sleep with Awaitility), order dependence (isolate/reset state), real clock (inject `Clock`), real network (WireMock/stubs), concurrency, randomness without a seed. **Don't `@Disabled` indefinitely** — flaky beats useless tests.

### Q12: Why does Mockito need `MockitoExtension` (or `MockitoAnnotations.openMocks`)?
**A:** It scans for `@Mock` and `@InjectMocks` annotations and instantiates the mocks before each test. Without it, the annotated fields are null. `@ExtendWith(MockitoExtension.class)` is the JUnit 5 way; `MockitoAnnotations.openMocks(this)` in `@BeforeEach` works without the extension.

### Q13: When is mocking a static method appropriate?
**A:** Almost never — it's a sign of a hard-to-test design. If you can refactor (inject a dependency, wrap the static call), do that. Mockito ≥ 3.4 supports `mockStatic` as an escape hatch when refactoring isn't possible.

### Q14: Coverage targets — what's enough?
**A:** Coverage is a hint, not a goal. 80–90% line coverage is typical for healthy codebases. The right question isn't "what's the number?" but "are the scenarios I'd be embarrassed to ship broken actually tested?"

### Q15: What's `ArgumentCaptor` for?
**A:** When the unit transforms input before passing it on, capture the argument actually sent to the mock and assert on it. `var c = ArgumentCaptor.forClass(Order.class); verify(repo).save(c.capture()); assertEquals("AAPL", c.getValue().symbol());`.

### Q16: Best-practice cheat list?
**A:** Test names describe behaviour. AAA structure. One logical concept per test. Test edge cases (null, empty, boundary). Independent — no shared mutable state. Don't test framework code. Don't mock value types or types you don't own (use real instances or fakes).

---

### Key Code Patterns

**Standard unit test**
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @InjectMocks OrderService service;
    @Test void placesOrder() {
        when(repo.save(any())).thenReturn(new Order("O1"));
        var r = service.placeOrder(new Order("O1"));
        assertEquals("O1", r.id());
        verify(repo).save(any());
    }
}
```

**Exception path**
```java
var ex = assertThrows(NotFoundException.class, () -> service.getOrder("X"));
assertEquals("not found: X", ex.getMessage());
```

**Spring controller slice**
```java
@WebMvcTest(OrderController.class)
class ControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;
    @Test void getOrder() throws Exception {
        when(service.getOrder("O1")).thenReturn(new Order("O1"));
        mvc.perform(get("/api/orders/O1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.id").value("O1"));
    }
}
```

**Argument captor**
```java
var captor = ArgumentCaptor.forClass(Order.class);
service.placeOrder(input);
verify(repo).save(captor.capture());
assertEquals("AAPL", captor.getValue().symbol());
```

---

## 8. Self-Test

**Easy**
- [ ] What's the testing pyramid?
- [ ] What does `@BeforeEach` do? When use `@BeforeAll`?
- [ ] How do you assert that a method throws?
- [ ] What's the difference between `@Mock` and `@InjectMocks`?

**Medium**
- [ ] Walk through the AAA structure of the §3 example.
- [ ] When is a `Spy` better than a `Mock`?
- [ ] Write a `@WebMvcTest` for `GET /api/orders/{id}` returning 200 and JSON.
- [ ] Why does constructor injection make a class easier to test?

**Hard**
- [ ] Argue for using a fake `InMemoryRepository` over a Mockito-mocked one in some tests.
- [ ] Why is mocking a static method a code smell? Show a refactor that removes the need.
- [ ] What makes a test flaky? List four common causes with fixes.
- [ ] Show how to verify the actual object passed to a mock when the unit transforms its input.
- [ ] Why does 100% line coverage not imply behaviour-tested?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Unit test** | One class in isolation, dependencies mocked. |
| **Integration test** | Multiple components wired together. |
| **E2E test** | The whole app, exercised through its outer interface. |
| **Mock** | A fake object whose behaviour you program. |
| **Spy** | A real object you can selectively override. |
| **Fake** | A real but simplified implementation (e.g. in-memory repo). |
| **Stub** | A mock that just returns canned values. |
| **AAA / Given-When-Then** | Arrange / Act / Assert — a unit test's three beats. |
| **`@Mock`** | Mockito annotation creating a mock instance. |
| **`@InjectMocks`** | Mockito annotation creating the unit-under-test with mocks injected. |
| **`@MockBean`** | Spring Boot annotation replacing a real bean with a Mockito mock in the loaded context. |
| **Argument captor** | Mockito tool to inspect actual args passed to a mock. |
| **Test slice** | Spring Boot annotation loading just one layer (`@WebMvcTest`, `@DataJpaTest`). |
| **Testcontainers** | Library that runs real services (DB, Kafka, …) in Docker for integration tests. |
| **Flaky test** | A test that passes inconsistently. Worse than no test. |
| **Coverage** | Percentage of lines / branches executed by tests. A floor, not a goal. |
| **TDD** | Red → Green → Refactor — write the test first. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/10_testing_doubts.md) · [← Prev: 09 SOLID](./09_solid.md) · [Next: 11 Spring Core →](./11_spring_core.md)

[↑ Back to top](#10--testing)
