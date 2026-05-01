# 10 — Testing

[← Back to Index](./00_INDEX.md) | **Priority: 🟢 Medium**

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
